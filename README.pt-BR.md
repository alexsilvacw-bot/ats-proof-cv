# ATS-Proof CV

[Read this guide in English](README.md)

A ATS-Proof CV adapta currículos com **busca vetorial local sobre o histórico profissional**, seguida de pontuação das evidências e revisão antes da geração do PDF.

## Busca vetorial e seleção de evidências

Cada requisito da vaga vira uma consulta. A skill gera embeddings com `text-embedding-3-large`, armazena os vetores da base em SQLite e calcula a similaridade de cosseno localmente. Uma busca lexical ponderada pela raridade dos termos complementa o ranking vetorial. Os resultados formam o conjunto de relatos que será avaliado.

O `gpt-6-luna` pontua cada relato recuperado de 1 a 5 por requisito. A seleção agrupa relatos da mesma realização dentro de um vínculo, prioriza as evidências mais fortes de cada requisito e usa MMR (Maximal Marginal Relevance) para escolher os complementos. O algoritmo favorece fatos novos e detalhes quantitativos relevantes e penaliza redundância. Um requisito composto pode ser sustentado por várias fontes. A revisão registra apoio direto, parcial, transferível ou ausente e só aceita alterações que preservem a melhor nota elegível já coberta por requisito.

## Revisão antes do PDF

Uma chamada separada ao modelo confere se o resumo e a ordem dos bullets refletem as responsabilidades centrais da vaga. O fluxo verifica se as métricas das fontes selecionadas foram preservadas e aplica correções apenas aos trechos sinalizados. Acréscimos de contexto ou responsabilidade ficam registrados para revisão factual humana.

## Custo da API

**A execução usa a API paga da OpenAI. O custo medido no Brasil foi de R$ 0,20 por currículo.** O total em reais varia conforme o consumo de tokens e o câmbio. [Preços da API da OpenAI](https://developers.openai.com/api/docs/pricing)

## Instalação

1. Baixe ou clone este repositório e abra a pasta `ats-proof-cv`. Para clonar pelo terminal, use `git clone https://github.com/alexsilvacw-bot/ats-proof-cv.git`.
2. Use Python 3.10 ou superior e LibreOffice no macOS ou Linux. Disponibilize `soffice` ou `libreoffice` no terminal. No Windows, use um ambiente Linux como WSL com as dependências instaladas nele.
3. Na pasta da skill, execute `python -m pip install -r requirements.txt` no seu ambiente Python. Se o seu sistema usa `python3`, substitua `python` nos comandos.
4. Configure `OPENAI_API_KEY` no ambiente que executará o script. A chave precisa ter acesso a `gpt-6.1-sol` para extração, `gpt-6-luna` para pontuação, revisão e redação, e `text-embedding-3-large` para os vetores. [Autenticação da API](https://developers.openai.com/api/reference/overview#authentication)
5. Para usar pelo Codex, copie a pasta da skill para `.agents/skills/` no seu projeto e invoque `$ats-proof-cv`. Para usar pelo terminal, execute os comandos abaixo a partir da pasta da skill. [Skills no Codex](https://developers.openai.com/codex/skills)

## Cadastrar e indexar suas experiências

- **No Codex.** Anexe seu currículo ou cole suas realizações e envie este pedido. «Use `$ats-proof-cv` para cadastrar minhas experiências em `dados/carreira.json`. Preserve os cargos, as datas e as métricas que forneci. Mostre a base para eu revisar antes de indexar.»
- **Manualmente.** Copie [a base fictícia](examples/carreira.json) para `dados/carreira.json` e preencha com os seus dados. O campo `contato` é opcional. Use um vínculo por combinação de empresa, cargo e período. Dentro de `experiencia`, inclua um item por relato. Mantenha os IDs de vínculos e relatos estáveis e distintos.

| Campo | Como preencher |
| --- | --- |
| `experiencias` | Lista de vínculos. Cada vínculo tem `id`, `empresa`, `cargo`, `periodo` e a lista `experiencia` |
| `periodo` | Use `AAAA-MM` para `entrada` e `saida`. Use `null` para mês desconhecido. Em um vínculo atual, marque `atual` como `true` e deixe a saída como `null` |
| `experiencia` | Cada relato tem `id` e `texto`. Descreva a ação, o contexto e o resultado, se conhecido. Preserve as unidades e os qualificadores dos números |

Revise o JSON e prepare a base. Guarde a base e os resultados na sua área de trabalho privada.

```bash
python scripts/cv.py index --base dados/carreira.json
```

Na preparação, a API da OpenAI recebe os relatos para organizar as realizações e criar os vetores. O índice e os dados operacionais ficam em `.ats-proof-cv/`. Ao acrescentar uma experiência, crie um novo ID. Ao corrigir um relato, mantenha seu ID. Execute `index` novamente após editar o arquivo. Os vetores de textos idênticos com o mesmo ID são reutilizados, e a organização das realizações é atualizada para a versão da base. Use `--workspace` com o mesmo caminho nos comandos se preferir outra pasta.

## Criar e retomar uma candidatura

Forneça o anúncio em TXT UTF-8 ou passe a URL pública com `--url`.

```bash
python scripts/cv.py create --vaga dados/vaga.txt
```

O caminho da candidatura aparece no terminal. Para retomar uma execução interrompida, passe esse mesmo caminho. O fluxo retoma a etapa pendente e pode reaproveitar respostas compatíveis do checkpoint temporário.

```bash
python scripts/cv.py resume --candidatura .ats-proof-cv/candidaturas/empresa_cargo
```

A seleção normalmente trabalha com um recorte da base. Se a busca já retornar todos os textos de uma base pequena, `--permitir-base-integral` autoriza a avaliação desse conjunto. Use a opção de forma explícita, pois nesse caso todos os relatos recuperados podem ser enviados à API para pontuação.

### Testar com os dados fictícios

Os dois comandos fazem chamadas à API e usam uma área de trabalho própria para o exemplo.

```bash
python scripts/cv.py index --base examples/carreira.json --workspace dados/demo
python scripts/cv.py create --vaga examples/vaga.txt --workspace dados/demo --permitir-base-integral
```

## Conferir o resultado

O fluxo gera o PDF em português ou inglês, conforme a vaga. O DOCX é o intermediário usado para a conversão. Os JSONs na pasta da candidatura registram o percurso.

- `requisitos-vaga.json` contém os requisitos extraídos e suas prioridades.
- `experiencias-aprovadas.json` contém a seleção e o mapa de cobertura por componente. O nome do arquivo identifica a seleção do processo. A aprovação factual é humana.
- `dados-reescritos.json` contém a redação, o parecer de alinhamento e os acréscimos pendentes de revisão.
- `origens-recuperacao.json` mostra de onde vieram as experiências recuperadas.

Abra o PDF e revise o conteúdo com os JSONs, especialmente as métricas, a atribuição dos resultados e os acréscimos pendentes. As listas de habilidades e ferramentas vêm da extração do anúncio e também precisam de conferência. A conversão e a validação de texto do PDF são locais. Durante a candidatura, o anúncio, os requisitos e as evidências pertinentes são enviados à OpenAI para as etapas de extração, embeddings, contagem de entrada, avaliação e redação. Nome e contato do cabeçalho são preenchidos localmente.

[Instruções operacionais da skill](SKILL.md)

[Licença MIT da edição pública](LICENSE)
