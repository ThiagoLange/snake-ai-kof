# Cobrinha IA — Q-learning em Kof

Uma IA que **aprende sozinha** a jogar o jogo da cobrinha, escrita 100% na
linguagem Kof — incluindo o algoritmo, o treino e o servidor web que
transmite a partida. Sem rede neural, sem biblioteca: Q-learning tabular.

Resultado típico: melhor de 30+ pontos, média ~11 nos últimos 100
episódios, 48/64 estados visitados. Cada treino é único: a seed vem do
relógio (`time.now()`), então a IA nunca morre no mesmo lugar duas vezes.
O log imprime a seed — para repetir um treino, fixe ela no `main()`.

## Rodar tudo (passo a passo)

Pré-requisito: Kof 0.4.10-beta instalado e no `PATH`.

```bash
export PATH="$HOME/.kof/bin:$PATH"
kof version   # deve mostrar: kof 0.4.10-beta
```

1. Entrar na pasta do projeto:

```bash
cd snake-ai-kof
```

2. Treinar a IA (cerca de 1 minuto, gera `watch.html` e `frames.json`):

```bash
kof run snake_ai.kof
```

3. Ver a IA jogar — opção A, arquivo direto (sem servidor):

```bash
xdg-open watch.html
```

4. Ver a IA jogar — opção B, servida pelo próprio Kof:

```bash
kof run server/server.kof
# abra http://localhost:8888 no navegador
```

5. Ver o treino ao vivo (a IA aprendendo em tempo real):

```bash
# terminal 1: servidor
kof run server/server.kof
# terminal 2: treino (escreve progress.jsonl a cada episódio)
kof run snake_ai.kof
```

Abra http://localhost:8888 e role até "Treino ao vivo": o gráfico
atualiza a cada segundo com episódio, placar e recorde enquanto o treino
roda no outro terminal.

A página mostra a cobrinha se movendo sozinha (play/pausa, reiniciar,
velocidade), placar e curva de aprendizado. Os dados da partida vêm de
`GET /api/frames` e o JS de `GET /app.js`, tudo servido pelo `kof.web`
(JS externo porque o CSP padrão do servidor bloqueia `<script>` inline).

Para ver a evolução do aprendizado, use o seletor "Evolucao": são 10
demos gravadas durante o treino (ep 250 até 2500). No começo a IA mal
sai do lugar; no fim ela atravessa a grade comendo.

6. Rodar os testes:

```bash
kof test tests/snake_test.kof   # 8 testes
kof check snake_ai.kof          # type-check
kof check server/server.kof     # type-check
```

## Como a IA funciona

- **Estado (6 bits, 64 estados):** perigo reto/direita/esquerda + comida
  frente/esquerda/direita, tudo relativo à direção da cabeça.
- **Ações relativas:** reto, virar à direita, virar à esquerda.
- **Recompensa:** +10 comer, −10 morrer, ±1 aproximar/afastar (Manhattan).
- **Hiperparâmetros:** 2500 episódios, grade 12×12, α=0.1, γ=0.9,
  ε 1.0→0.05 (decaimento 0.996), RNG próprio (LCG mod 65521) com seed do
  relógio. Para mudar os episódios, edite `var episodes` no `main()`.

## Estrutura

```
snake-ai-kof/
├── snake_ai.kof          # jogo + Q-learning + treino + export HTML/JSON
├── server/server.kof     # front 100% em Kof (kof.web: / e /api/frames)
├── tests/snake_test.kof  # 8 testes (RNG, estado, comer, morte, Q)
├── watch.html            # gerado: replay standalone da partida
├── frames.json           # gerado: frames + scores para a API
├── progress.jsonl        # gerado: 1 linha por episódio para o ao vivo
└── README.md
```

`tests/` e `server/` ficam em subdirs porque o `kof run` compila o
diretório inteiro como pacote (`PKG005`).

## Notas de Kof

- Sem classes: só funções top-level sobre listas paralelas
  (`box = [dir,foodX,foodY,score,over,steps,w,h]`), pois classe com campos
  `List<Int>` + função top-level com `%` sobre chamada quebra o backend
  JVM com `VerifyError` (bug reportado upstream).
- RNG próprio (LCG mod 65521) para treino 100% reprodutível.
- Servidor usa `headerSet("Content-Type", "text/html")` — sem isso o
  navegador recebe `text/plain` e não renderiza.
