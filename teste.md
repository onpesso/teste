# Job Matcher: Automatizando a Caça de Vagas com TF-IDF e LLM Local

*18 de março de 2026 · 💬 Participe da Discussão*

---

**On this page**

- O Problema: Procurar Vaga em 2026
- A Decisão de Stack
- O Glassdoor Problem: Quando o Scraper Quebra
- A Hierarquia de Confiabilidade na Prática
- O Parser de Currículo: Cache por Hash
- O Motor de Match: Duas Velocidades
- A Fórmula do Score Final
- Deduplicação e o Problema de Concorrência
- Números
- O que Não Está Aqui
- Conclusão: A Automação Que Deveria Existir

---

## O Problema: Procurar Vaga em 2026

Tem uma coisa que todo desenvolvedor brasileiro faz de forma profundamente ineficiente: procurar emprego.

O fluxo padrão é assim: você abre o LinkedIn, digita "desenvolvedor Python", filtra por São Paulo, vê 347 vagas. Abre a primeira, lê, fecha. Abre a segunda, lê. A terceira parece boa mas pede Oracle e você faz cinco anos que não toca Oracle. A quarta fica confusa porque mistura "Python" com "Java" no mesmo parágrafo e você não consegue saber qual é o foco real. Você vai fazendo isso por quarenta minutos, abre o Indeed também, encontra as mesmas vagas com outros nomes, vai no Glassdoor, mais quarenta minutos.

Duas horas depois você se candidatou pra três vagas e não tem certeza se elas eram boas ou se você simplesmente ficou cansado de ler.

Isso é um problema de automação com cara de 2015. A diferença é que em 2015 você teria dificuldade em fazer o scoring de relevância com qualidade. Em 2026 você tem TF-IDF, cosine similarity, e um LLM local no Ollama que lê currículo e descrição de vaga e te diz em português por que combina ou não.

Foi isso que o Job Matcher resolve. Você roda um comando, ele faz o trabalho chato, te entrega uma tabela ordenada por score.

---

## A Decisão de Stack

Antes de qualquer coisa: por que Python?

Já entrei nessa discussão antes — Crystal pra CLIs, Rust pra performance crítica, etc. Mas aqui a decisão foi direta. O `python-jobspy` é a única biblioteca open-source com suporte real a LinkedIn, Indeed e Glassdoor. Tem 2.9k estrelas, está sendo mantida, e a alternativa seria escrever do zero o reverse engineering de cada site. Não é um projeto que vale isso.

O resto do stack seguiu a lógica do menor atrito: SQLAlchemy + PostgreSQL + Alembic porque migrações versionadas são inegociáveis em qualquer projeto que cresce, scikit-learn porque TF-IDF cosine similarity são quatro linhas de código com ele, Typer + Rich porque CLI sem output bonito é CLI que você abandona.

A arquitetura ficou assim:

```
resume/meu_curriculo.tex
    └─▶ resume_parser.py  ──▶  resume_profiles (DB, cache MD5)
                                      │
LinkedIn / Indeed / Glassdoor         │
    └─▶ scraper.py  ──────▶  jobs (DB, dedup por job_url)
                                      │
matcher.py  ◀─────────── lê ambos ───┘
    ├── keyword_score:  TF-IDF cosine similarity
    └── ai_score:       Ollama ou Copilot proxy → JSON
    └──▶  match_results (DB)

cli.py (Typer) + scheduler.py (APScheduler) orquestram tudo
```

Nenhuma surpresa arquitetural. O interessante está nos detalhes de implementação.

---

## O Glassdoor Problem: Quando o Scraper Quebra

Quando o `python-jobspy` diz que suporta Glassdoor, o que ele quer dizer é que *suportava*. A API GraphQL que o jobspy usava para o Glassdoor retorna 403 consistentemente. A equipe do jobspy abriu issues sobre isso. Não é bug de implementação — é bloqueio deliberado.

A primeira reação de qualquer um é: "ok, vou usar BeautifulSoup no HTML". Essa reação está errada, e eu já escrevi sobre isso antes no post de [Web Scrapping em 2026](https://www.akitaonrails.com/2026/02/18/web-scrapping-em-2026-bastidores-do-the-m-akita-chronicles/).

O Glassdoor em 2026 é uma SPA Next.js com RSC (React Server Components). Os nomes de classe CSS são gerados pelo Tailwind JIT a cada deploy. A hierarquia DOM muda a cada redesign. Qualquer `soup.find('div', class_='...')` que você escrever hoje é uma bomba-relógio com fusível de semanas.

Existe uma lei não-escrita de scraping que aprendi da forma mais dolorosa possível: **você identifica conteúdo pelo comportamento, não pelo nome**.

---

## A Hierarquia de Confiabilidade na Prática

Antes de qualquer implementação, precisei entender o que o Glassdoor *realmente* expõe e o que é estável. A hierarquia que uso:

| Fonte | Confiabilidade | Manutenção |
|-------|---------------|------------|
| Dados estruturados (RSC stream, JSON-LD, `__NEXT_DATA__`) | Alta | Baixa |
| Seletores semânticos (href patterns, `<article>`, `<time>`) | Média | Média |
| Heurísticas estruturais (URL length, text clustering) | Média | Média |
| Seletores CSS / hierarquia DOM | Baixa | Altíssima |

O Glassdoor pós-Next.js tem três coisas estáveis:

**1. O RSC stream.** Uma página de busca retorna ~840KB de HTML com um payload de React Server Components embutido em tags `<script>`. Esse payload contém `"jobListings":[...]` com título, empresa, localização, `listingId` e snippet de *todos* os resultados — dados limpos, tipados, gerados pelo framework antes de virar HTML.

**2. O JSON-LD.** `<script type="application/ld+json">` com um `ItemList` contendo as URLs dos trinta primeiros resultados, cada uma com `?jl=<listingId>`. Estável porque é gerado para SEO/schema.org — o time de frontend não mexe nisso.

**3. A lógica de URL.** URLs de vagas são sempre mais longas que URLs de navegação. Isso é arquitetural.

A implementação do `glassdoor_scraper.py` usa exatamente essa hierarquia. **Máximo 2 requisições HTTP por execução**: uma busca geral que extrai o RSC stream, e eventualmente uma requisição de enriquecimento para descrição completa quando o snippet está truncado. Cruzamos `listingId` do RSC com `?jl=` do JSON-LD para montar as URLs reais.

```python
# O que NÃO usamos (e por quê):
#   - BeautifulSoup em class names → muda a cada deploy com Tailwind JIT
#   - _find_card_ancestor → sobe na árvore DOM, quebra a cada redesign
#   - data-test attributes → ligeiramente mais estáveis, mas não confiáveis
#
# O que usamos:
#   Prioridade 1: RSC stream → dados estruturados do framework
#   Prioridade 2: JSON-LD → semântico, gerado para SEO, estável
```

Rate limiting rigoroso: curl_cffi com Chrome124 impersonation, 8-15 segundos de delay aleatório entre requests de enriquecimento. Se bloqueado (403/captcha), retorna os dados parciais que já coletou — nunca crasha, nunca levanta exceção para o chamador.

---

## O Parser de Currículo: Cache por Hash

O parser tem uma propriedade que deveria ser óbvia mas frequentemente é ignorada: **re-parsear um arquivo que não mudou é desperdício**.

A solução é calcular o MD5 do arquivo `.tex` e comparar com o hash armazenado no banco. Cache hit → retorna o perfil existente em microsegundos. Cache miss → parseia e persiste.

```python
def get_or_parse_resume(path: str | Path, db: Session) -> ResumeProfile:
    file_hash = md5_file(path)

    existing = (
        db.query(ResumeProfile)
        .filter(
            ResumeProfile.file_path == str_path,
            ResumeProfile.file_hash == file_hash,
        )
        .first()
    )
    if existing:
        return existing  # cache hit

    data = parse_latex_file(path)
    profile = ResumeProfile(file_hash=file_hash, ...)
    db.add(profile)
    db.flush()  # caller controla a transação
    return profile
```

O `db.flush()` sem commit é deliberado: o chamador controla quando a transação fecha. Isso permite compor operações sem commits intermediários — se o parse passar mas o que vier depois falhar, o rollback desfaz tudo junto.

O parser usa dois níveis para extrair informação do LaTeX:

- **`pylatexenc`** — converte toda a sintaxe LaTeX em texto Unicode limpo. Lida com macros, ambientes customizados, caracteres especiais.
- **`TexSoup`** — parseia a *estrutura* do documento. Navega por `\section` e `\subsection`, extrai o conteúdo de cada seção com semântica preservada.

Para extração de skills, mantemos um vocabulário curado de ~120 tecnologias e geramos um regex com word-boundary para match case-insensitive. Simples, determinístico, sem NLP. Se "FastAPI" não aparece no seu currículo, não está lá — o sistema não infere.

---

## O Motor de Match: Duas Velocidades

O sistema tem dois modos de scoring com casos de uso diferentes.

**`--keyword-only`** — Sem LLM. TF-IDF cosine similarity entre texto do currículo e descrição da vaga. Usa bigramas (`ngram_range=(1, 2)`) para capturar termos compostos como "machine learning", TF sublinear (`sublinear_tf=True`) para não deixar um termo frequente dominar o vetor, e sem stop words porque CVs em português brasileiro + inglês técnico misturam os dois idiomas e stop words em inglês vão remover termos errados.

```python
vectorizer = TfidfVectorizer(
    analyzer="word",
    token_pattern=r"(?u)\b\w[\w+#.-]*\b",  # mantém C++, .NET, scikit-learn
    stop_words=None,       # multilingual — sem stop words
    ngram_range=(1, 2),    # unigrams + bigrams
    sublinear_tf=True,     # log(1+tf) amortece termos muito frequentes
)
tfidf_matrix = vectorizer.fit_transform([resume_text, job_description])
score = cosine_similarity(tfidf_matrix[0:1], tfidf_matrix[1:2])[0][0]
```

Resposta em milissegundos. Bom para filtrar ruído quando você tem 200 vagas novas e quer descartar as claramente irrelevantes antes de rodar o LLM.

**`--with-ai`** — Com LLM. Manda currículo + vaga para Ollama (local, offline) ou um Copilot proxy (endpoint Anthropic-compatível). O modelo lê os dois, raciocina sobre compatibilidade semântica — entende que "engenharia de dados" no currículo tem a ver com "pipeline ETL" na vaga, que "Python" implica o ecossistema em volta — e devolve um JSON estruturado.

O prompt especifica contrato estrito: **apenas JSON, sem markdown, sem texto adicional**.

```
Responda APENAS com um objeto JSON válido, sem texto adicional, sem markdown.
{
  "score": <número inteiro de 0 a 100>,
  "matched_skills": ["skill1", "skill2"],
  "missing_skills": ["skill1", "skill2"],
  "explanation": "Justificativa em 2-3 frases em português."
}
```

A explicação em português é intencional. Quando você está lendo 50 resultados num painel, você lê mais rápido no idioma em que pensa.

O parser de resposta tenta parse direto primeiro, depois fallback com regex `\{.*\}` caso o modelo envolva o JSON em markdown (isso acontece, especialmente com modelos menores). Score é clampeado em [0, 100] independente do que o modelo retornar — alucinações de valor fora do range são reais.

---

## A Fórmula do Score Final

```python
def compute_final_score(keyword_score: float, ai_score: Optional[float]) -> float:
    if ai_score is not None:
        return round(0.4 * keyword_score + 0.6 * ai_score, 2)
    return round(keyword_score, 2)
```

60% para o LLM, 40% para keywords. TF-IDF é literal — não entende sinônimos, não entende contexto, não sabe que "Engenheiro de Dados" e "Data Engineer" são a mesma coisa. O LLM entende, mas pode alucinar. O peso maior para o LLM com um multiplicador conservador para o método determinístico é uma escolha deliberada de design: queremos que o contexto semântico domine, mas que o sinal de keywords seja um freio contra alucinações.

Se o LLM estiver inacessível (Ollama não rodando, Copilot proxy com timeout), o sistema cai graciosamente para keyword-only. `ai_score` vira `None`, `final_score = keyword_score`. Nenhuma exceção, nenhum crash — dados parciais são preferíveis a falha total.

---

## Deduplicação e o Problema de Concorrência

Qualquer sistema de scraping periódico vai re-encontrar as mesmas vagas. A deduplicação por URL precisa acontecer no banco, não na aplicação.

```sql
INSERT INTO jobs (id, job_url, title, ...)
VALUES (:id, :job_url, :title, ...)
ON CONFLICT (job_url) DO NOTHING
```

Parece trivial. Mas tem uma implicação importante: se dois processos de scraping rodarem simultaneamente (o APScheduler pode disparar antes do anterior terminar, especialmente com buscas longas no Glassdoor), ambos vão tentar inserir a mesma URL. Com `ON CONFLICT DO NOTHING` no banco, um dos dois descarta silenciosamente. Com checagem na aplicação (`if url not in existing_urls`), você tem uma race condition clássica — os dois processos leram o banco antes de qualquer um ter inserido, então ambos acham que o job é novo.

O campo `job_url` tem restrição `UNIQUE` no banco. Essa é a única fonte de verdade.

Pelas mesmas razões, nenhuma migration usa `Base.metadata.create_all()`. Cada mudança de schema tem uma migration Alembic versionada. Em produção, `create_all()` é uma forma eficiente de perder controle do histórico do banco.

---

## Números

```
Módulo                          Linhas
────────────────────────────────────────
src/glassdoor_scraper.py          1007
src/graphql_schema.py              733
src/api.py                         556
src/cli.py                         449
src/scraper.py                     280
src/matcher.py                     311
src/resume_parser.py               198
src/scheduler.py                    99
(outros)                           447
────────────────────────────────────────
Total src/                        4080

tests/test_glassdoor_scraper.py    876
tests/test_scraper.py              183
tests/test_matcher.py              169
tests/test_resume_parser.py        144
────────────────────────────────────────
Total tests/                      1429
```

O `glassdoor_scraper.py` ter 1007 linhas e 876 linhas de teste não é acidente. O Glassdoor é o componente mais frágil do stack — qualquer mudança no Next.js deles pode quebrar a extração do RSC stream sem aviso. A cobertura de testes alta é o que torna possível detectar regressões sem precisar debugar ao vivo contra o Cloudflare. Cada request desperdiçado em debug queima rate-limit e pode resultar em ban de IP por minutos ou horas.

A regra operacional de debug que segui: antes de cada alteração no scraper, salve o HTML/response localmente, analise offline, forme uma hipótese clara do que está errado e *por quê*, faça uma mudança cirúrgica, teste. Nunca empilhar cinco mudanças esperando que alguma funcione.

---

## O que Não Está Aqui

Algumas decisões de escopo valem mencionar explicitamente.

**Sem Selenium.** O README lista GeekHunter e Remotar com Selenium para a Fase 2. Selenium é caro em manutenção, precisa de Chrome instalado, é lento, e quebra com atualizações de webdriver. A regra é: só vá para headless browser quando não há alternativa. Para LinkedIn e Indeed, o jobspy funciona. Para Glassdoor, o RSC stream funciona. Selenium seria o último recurso.

**Sem inferência de skills por NLP.** O sistema não tenta concluir que você sabe Django logo você sabe Python. O vocabulário de skills é curado e o match é direto. Falso positivo no score de compatibilidade é pior que falso negativo — você vai se candidatar para vaga errada.

**Dois providers de LLM.** Ollama para quem quer 100% offline e privado (llama3, mistral). Copilot proxy para quem já tem acesso ao Copilot e não quer rodar modelo local. Trocar entre eles é setar `AI_PROVIDER` no `.env`. O código não assume qual está disponível.

---

## Conclusão: A Automação Que Deveria Existir

Tem uma categoria de projetos que eu chamo de "deveria existir mas ninguém fez direito". Data mining de vagas com scoring automático é um deles. As soluções que existiam ou dependiam de APIs pagas, ou eram scripts descartáveis sem banco de dados, ou faziam scraping frágil que quebrava em semanas.

O Job Matcher não é um script. É um sistema com banco de dados, migrações versionadas, cache de currículo por hash, deduplicação no nível de banco, dois modos de scoring com fallback gracioso, e um scraper de Glassdoor que entende a diferença entre o que é estável e o que é cosmético num site Next.js.

O que aprendi no processo:

1. **Scraping em 2026 é uma guerra de atrito**. Glassdoor, LinkedIn, Indeed — todos têm anti-bot crescente. A única estratégia sustentável é usar fontes de dados estruturados quando existem (RSC stream, JSON-LD, feeds) e evitar depender de qualquer coisa que o time de frontend possa mudar sem você saber.

2. **LLM local muda o que é viável fazer**. Antes, "analisar semanticamente 200 descrições de vaga" custaria dinheiro de API ou seria muito lento. Com Ollama e um modelo razoável rodando localmente, é uma decisão de tempo de CPU, não de orçamento.

3. **O banco de dados é a fonte de verdade, não a aplicação**. Deduplicação por URL no PostgreSQL com `ON CONFLICT`, hash de arquivo no `resume_profiles`, migrações Alembic para cada mudança de schema. A aplicação pode ter bugs; o banco mantém a consistência.

```bash
pip install -e ".[dev]"
alembic upgrade head
ollama pull llama3

python -m job_matcher parse-resume resume/meu_curriculo.tex
python -m job_matcher search "python developer" --location "Brazil"
python -m job_matcher match --with-ai
python -m job_matcher list --min-score 60 --sort score
```

O repositório está no GitHub. MIT License.

---

*Powered by python-jobspy · scikit-learn · Ollama · SQLAlchemy · Typer · Rich*
