# AllasCode Qwen Coder Distillation — TypeScript + Zig MoE

Projeto para destilar comportamento de `Qwen/Qwen2.5-Coder-7B-Instruct` para um student menor especializado em TypeScript e Zig, com uma taxonomia compartilhada de especialistas.

## Manual

Leia o [Manual de destilação](MANUAL-DESTILACAO.md) antes de executar os notebooks. Ele documenta o pipeline de dados, validação, treino, avaliação, limites de memória e os requisitos para implementar um MoE real.

## Notebooks

- [00 — Pipeline geral para Google Colab](notebooks/00-colab-distillation.ipynb)
- [01 — Destilação para TypeScript](notebooks/01-typescript-distillation.ipynb)
- [02 — Destilação para Zig](notebooks/02-zig-distillation.ipynb)

Os três notebooks usam exatamente os mesmos IDs de especialistas. Os notebooks de linguagem apenas filtram os exemplos pela linguagem.

## Especialistas compartilhados

| ID canônico | Área de especialização |
|---|---|
| `syntax_repair` | Correção de erros sintáticos |
| `logic_repair` | Correção de erros lógicos |
| `memory_safety` | Memória, recursos e segurança de acesso |
| `strict_semantic_typing` | Tipagem semântico-nominal estrita e forte |
| `test_engineering` | Geração e execução de testes |
| `communication_channels` | Canais de comunicação, protocolos e serialização |
| `zero_trust_security` | Zero trust e segurança |
| `performance_optimization` | Otimização de desempenho |

## Distinção técnica importante

Os notebooks oferecem uma baseline executável de **destilação comportamental via SFT/LoRA** para um student menor. Isso, por si só, não cria uma arquitetura Mixture of Experts: LoRA é uma técnica de adaptação de pesos. Um MoE real exige experts distintos e um router/gating treinável, além de métricas de balanceamento e roteamento. Essa etapa está explicitamente descrita no manual e não deve ser confundida com o fine-tuning inicial.

## Dataset

JSONL com os campos obrigatórios `language`, `expert`, `instruction`, `input` e `output`. Use código próprio, sintético ou com licença compatível; valide compilação e testes antes de incluir exemplos. Nunca inclua segredos, tokens ou dados pessoais.

## Compatibilidade e validação

- TypeScript: fixe a versão de Node.js/TypeScript e execute compilador, lint e testes.
- Zig: fixe a versão do compilador e execute `zig fmt --check`, build e testes.
- Execute código gerado em sandbox sem rede, com limites de recursos.
- Avalie cada especialista e linguagem separadamente em um benchmark não visto.

Ver [MANUAL-DESTILACAO.md](MANUAL-DESTILACAO.md) para critérios de aceite, limitações de Colab e observações de licença.
