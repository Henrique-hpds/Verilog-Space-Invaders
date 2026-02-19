# Space Invaders na FPGA - MC613

Implementacao de um jogo 2D em Verilog para placa com FPGA Cyclone V, com renderizacao em VGA, inimigos em formacao, disparo continuo da nave e sistema de vidas/pontuacao.

## Visao geral

O projeto implementa um jogo no estilo arcade:

- A nave do jogador se move horizontalmente.
- A nave dispara projeteis para cima.
- Inimigos se movem em grupo, descem ao tocar nas bordas e podem eliminar o jogador por colisao.
- Cada inimigo pode disparar projeteis em momentos pseudoaleatorios.
- O jogo termina por derrota (sem vidas ou colisao direta) ou vitoria (eliminar todos os inimigos ativos).

## Hardware-alvo

- FPGA family: `Cyclone V`
- Device: `5CSEMA5F31C6`
- Projeto Quartus: `projeto.qpf` / `projeto.qsf`
- Entidade de topo: `projeto`

## Entradas e saidas

Mapeamento principal definido em `src/projeto.v`:

- `SW[0]`: reset global
- `SW[1]`: pausa (quando ligado, congela a movimentacao)
- `KEY[0]`: mover nave para a direita
- `KEY[1]`: mover nave para a esquerda
- `VGA_*`: video (RGB 8 bits por canal + sinais de sincronismo)
- `HEX0`: exibicao de pontuacao em 7 segmentos
- `LEDR[1:0]`: vidas restantes

Observacoes:

- As teclas sao ativas em nivel baixo e passam por tratamento basico de debounce em `src/keys.v`.
- `KEY[2]` e `KEY[3]` sao lidas, mas nao usadas na logica atual.

## Arquitetura

### Topo e integracao

- `src/projeto.v` conecta:
  - logica do jogo (`entities`)
  - gerador VGA (`vga`)
  - renderizador (`tela`)
  - entrada de teclas (`keys`)
  - decodificador para 7 segmentos (`cb7s`)

### Logica do jogo

- `src/entities.v`: integra nave, frota e projetil aliado.
- `src/nave.v`: movimentacao da nave + estado de derrota por vidas.
- `src/frota.v`: agrega fileiras de inimigos e consolida estado global.
- `src/fileira.v`: instancia inimigos da fileira e controla troca de sentido.
- `src/inimigo.v`: movimentacao individual, colisao com tiro aliado e colisao com nave.
- `src/bolainimiga.v`: projetil inimigo com disparo pseudoaleatorio via `lfsr`.
- `src/bola.v`: projetil aliado (movimento vertical continuo).
- `src/lfsr.v`: gerador pseudoaleatorio usado nos disparos inimigos.

### Video e sprites

- `src/vga.v`: temporizacao de 640x480 em area ativa (janela posicionada em relacao ao timing total).
- `src/tela.v`: composicao de cena (fundo, nave, inimigos, tiros, coracoes, telas de fim).
- `src/buffer.v`: leitura de bitmaps 1-bit por canal para desenhar sprites escaladas.
- `src/bufferFileira.v`: modulo auxiliar de desenho para conjuntos de objetos.

## Estrutura de pastas

- `src/`: modulos Verilog principais.
- `python/`: utilitarios para converter imagens em mapas binarios (`R_out.txt`, `G_out.txt`, `B_out.txt`).
- `imagem/`: arte auxiliar em formato texto.
- `output_files/`: arquivos gerados de compilacao (inclui `projeto.sof`).
- `stp/`: arquivos SignalTap.
- `documentacao/`: manuais e referencias da placa/dispositivos.
- `base/`: versao/base anterior do projeto.

## Como compilar e gravar

### Opcao 1: Quartus (GUI)

1. Abra `projeto.qpf` no Quartus Prime.
2. Confirme `projeto` como top-level entity.
3. Rode `Processing -> Start Compilation`.
4. Abra o `Programmer`, selecione o hardware, carregue `output_files/projeto.sof`.
5. Programe a FPGA.

### Opcao 2: Linha de comando

```bash
quartus_sh --flow compile projeto
```

Ao final, use o `Programmer` para gravar o `.sof` gerado.

## Fluxo para atualizar sprites

1. Coloque a imagem de entrada em `python/` (ou use caminho absoluto).
2. Rode o script:

```bash
cd python
python3 images.py
```

3. Informe o caminho da imagem quando solicitado.
4. Use os arquivos `R_out.txt`, `G_out.txt`, `B_out.txt` para atualizar os buffers binarios nos modulos de renderizacao (ex.: `src/tela.v`).

## Estado atual e observacoes

- O jogo usa 10 inimigos ativos na configuracao atual da frota.
- A pontuacao exibida em `HEX0` corresponde ao numero de inimigos eliminados.