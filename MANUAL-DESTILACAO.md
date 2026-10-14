# Manual de destilação: Qwen2.5-Coder 7B → MoE compacto para TypeScript e Zig

## 1. Objetivo e limites

Este projeto usa `Qwen/Qwen2.5-Coder-7B-Instruct` como **teacher** para produzir exemplos de alta qualidade e destilar comportamento para um student menor, especializado em TypeScript e Zig. O objetivo final é um modelo Mixture of Experts (MoE) com especialistas compartilhando a mesma taxonomia nos dois ecossistemas.

**Importante:** reduzir um modelo denso de 7B parâmetros não o transforma automaticamente em MoE. Os notebooks incluídos implementam o pipeline reproduzível de geração/curadoria de dados e fine-tuning supervisionado de um student menor. Isso é uma baseline de destilação comportamental, não uma conversão automática da arquitetura para MoE. Um MoE real requer experts separados, router/gating treinável, balanceamento de carga e avaliação do roteamento; pode ser implementado depois com uma arquitetura compatível ou código/modelo customizado. Não rotule um student denso como MoE.

## 2. Especialistas canônicos compartilhados

Os mesmos especialistas e IDs devem existir nos datasets TS e Zig. Cada exemplo recebe exatamente um especialista primário; opcionalmente, registre especialistas secundários em metadados. Os exemplos devem ser separados por linguagem sem alterar a taxonomia.

| ID | Especialista | Escopo |
|---|---|---|
| `syntax_repair` | Correção sintática | Parser errors, delimitadores, imports, gramática, versões da linguagem |
| `logic_repair` | Correção lógica | Invariantes, fluxo de controle, condições de borda, algoritmos |
| `memory_safety` | Memória e recursos | Ownership/lifetimes quando aplicável, alocação, leaks, bounds, recursos e estado |
| `strict_semantic_typing` | Tipagem semântico-nominal estrita e forte | Tipos explícitos, contratos, nomes semânticos, invariantes de domínio, evitar coerções inseguras |
| `test_engineering` | Geração e execução de testes | Unit/property/regression tests, casos-limite, testes reproduzíveis e análise de falhas |
| `communication_channels` | Canais de comunicação | Protocolos, serialização, concorrência, mensagens, timeouts e tratamento de falhas |
| `zero_trust_security` | Zero trust e segurança | Validação de entrada, autorização, limites de confiança, criptografia e mitigação de vulnerabilidades |
| `performance_optimization` | Otimização | Profiling, complexidade, alocações, latência e throughput sem regressões semânticas |

Essa taxonomia organiza o resumo atual do README em áreas distintas e estáveis. Se o projeto já definir nomes adicionais em arquivos futuros, mantenha-os compatíveis por meio de IDs canônicos versionados; não altere IDs silenciosamente.

## 3. Pipeline recomendado

1. **Defina versões e proveniência.** Registre commit do dataset, versões de Python/Transformers, GPU, modelo teacher/student, licença e hashes.
2. **Monte um corpus legal e seguro.** Use código próprio, código com licença compatível ou exemplos sintéticos. Remova credenciais, tokens, dados pessoais e segredos. Não use código de terceiros sem verificar a licença.
3. **Crie tarefas por especialista.** Inclua código defeituoso, diagnóstico, correção, explicação curta e testes. Misture exemplos positivos, negativos e casos-limite. Evite que o teacher invente resultados de compilação.
4. **Gere candidatos com o teacher.** Use temperatura baixa/moderada e prompts que exijam saída estruturada. Armazene prompt, resposta, linguagem, especialista, versão do modelo e seed quando disponível.
5. **Valide automaticamente.** TypeScript: `tsc --noEmit`, ESLint e testes isolados. Zig: use a versão alvo explicitamente, `zig fmt --check`, compilação e testes. Execute código não confiável em sandbox sem rede, com limites de CPU/memória/tempo.
6. **Filtre e revise.** Rejeite código que não compila, testes falsos, respostas fora do escopo, APIs inexistentes, vazamento de segredos e duplicatas. A validação do compilador não prova correção lógica.
7. **Divida o dataset sem vazamento.** Separe treino/validação/teste por projeto ou família de problema, não apenas por linhas aleatórias. Mantenha benchmark final intocado.
8. **Destile para o student.** O notebook usa SFT de respostas curadas como abordagem acessível em Colab. A destilação por logits exige acesso às distribuições do teacher e alinhamento de tokenizer; não é realizada por simples geração de texto.
9. **Avalie cada especialista e linguagem.** Reporte taxa de compilação, testes aprovados, correção semântica revisada, taxa de abstenção, latência, VRAM e regressões. Compare teacher, student e baseline.
10. **Somente então implemente MoE.** Crie experts especializados e router treinável, defina top-k, balanceamento e fallback. Treine/ajuste o router com rótulos de especialistas e meça roteamento errado, carga por expert e custo real de inferência.

## 4. Formato de dados

JSONL, um objeto por linha:

```json
{"language":"typescript","expert":"syntax_repair","instruction":"Corrija o erro e explique a mudança.","input":"const n: number = 'x';","output":"...","tests":"...","source":"synthetic-v1","split":"train"}
{"language":"zig","expert":"memory_safety","instruction":"Analise o gerenciamento de recursos e proponha correção.","input":"...","output":"...","tests":"...","source":"synthetic-v1","split":"train"}
```

Campos obrigatórios: `language`, `expert`, `instruction`, `input`, `output`. Valide `expert` contra os oito IDs canônicos. Não misture exemplos de teste no treino. Preserve a versão de Zig e TypeScript usada em cada amostra.

## 5. Execução no Google Colab

1. Abra o notebook adequado em `notebooks/` usando Colab (Arquivo → Abrir notebook → GitHub) ou faça upload do `.ipynb`.
2. Em Ambiente de execução → Alterar tipo de ambiente, selecione GPU. Uma GPU T4 pode não comportar todas as configurações; reduza sequência, batch e dataset.
3. Execute as células na ordem. Faça login no Hugging Face apenas se o modelo/dataset exigir autorização; nunca coloque tokens no notebook compartilhado.
4. Aponte `DATA_PATH` para JSONL já validado. Os notebooks não assumem que dados sintéticos gerados automaticamente sejam verdadeiros: compile e teste antes do treino.
5. Comece com poucas centenas de exemplos para validar o pipeline. Para treino sério, aumente o corpus balanceado por especialista e use validação independente.
6. Salve checkpoints e métricas em armazenamento persistente (por exemplo, Google Drive privado ou Hugging Face com acesso apropriado). O disco temporário do Colab pode ser apagado.

## 6. Student, memória e parâmetros

A configuração inicial usa um student instruct menor e quantização/LoRA quando suportados. O teacher de 7B é usado principalmente para criar respostas-alvo offline; carregar teacher e student simultaneamente pode exceder a VRAM. Não presuma que uma T4 consegue treinar um MoE completo.

- Comece com sequence length 1024; aumente apenas após medir memória.
- Use micro-batch 1 e gradient accumulation para reduzir VRAM.
- Use LoRA/QLoRA para adaptação econômica, quando compatível com o modelo.
- Mantenha checkpoints, logs e dataset fora da sessão efêmera.
- Fixe versões das dependências e faça um teste pequeno antes do treino completo.

## 7. Critérios de aceite

- Todos os exemplos passam validação de schema e usam um ID de especialista canônico.
- Sem duplicatas entre treino/validação/teste e sem segredos detectados.
- TypeScript compila com versão e configuração registradas; testes executam em sandbox.
- Zig formata, compila e executa testes na versão alvo registrada.
- Métricas reportadas por linguagem e especialista, não apenas média global.
- Comparação contra baseline e teacher em benchmark não visto.
- Para alegar MoE: demonstrar experts distintos, router/gating efetivo, distribuição de tokens por expert e comparação de latência/memória com baseline denso.

## 8. Estrutura dos notebooks

- `notebooks/00-colab-distillation.ipynb`: pipeline compartilhado e configuração.
- `notebooks/01-typescript-distillation.ipynb`: dataset, validações e treino para TypeScript.
- `notebooks/02-zig-distillation.ipynb`: dataset, validações e treino para Zig.

Os três usam a mesma lista de especialistas. O notebook de linguagem filtra os dados pela linguagem, não cria uma taxonomia separada.

## 9. Licenças e segurança

Confira a licença do teacher, do student, dos dados e de cada fonte de código antes de publicar pesos ou datasets. A licença de um modelo-base pode impor obrigações próprias. Nunca execute código gerado sem isolamento. Mantenha uma trilha de auditoria para prompts, respostas, filtros, versões e resultados dos testes.
