---
layout: post
title: "Do triângulo ao pixel: uma viagem pela pipeline gráfica"
date: 2026-10-29
---

Quando um jogo roda a 60 FPS, o computador tem cerca de **16,7 milissegundos** para transformar uma cena 3D inteira (objetos, luzes, materiais e uma câmera virtual) em uma única imagem 2D na tela. E depois fazer tudo de novo, sessenta vezes por segundo. Quem faz esse trabalho é a **pipeline gráfica**, que na aula de hoje vimos ser o verdadeiro "motor" dos gráficos em tempo real.

## Por que "pipeline"?

A palavra vem de qualquer sistema dividido em estágios por onde algo flui: um oleoduto, uma linha de montagem (como a do Chaplin em *Tempos Modernos*) ou o metrô. O exemplo do metrô deixa a ideia bem clara: o trem que está no Paraíso não pode seguir para a Vergueiro enquanto o trem da Vergueiro não avançar para São Joaquim. Cada estágio trabalha em paralelo com os outros, mas **um depende do anterior para continuar**.

Na computação gráfica, o que flui pela pipeline são os dados da cena, e o que sai no final é uma imagem. Conceitualmente, ela tem quatro estágios:

<div style="display:flex;flex-wrap:wrap;gap:6px;align-items:center;margin:1.2em 0;font-size:14px;">
  <div style="padding:8px 12px;border-radius:6px;background:#e8f0fe;border:1px solid #9bb7ef;"><b>Aplicação</b><br><small>CPU</small></div>
  <span>→</span>
  <div style="padding:8px 12px;border-radius:6px;background:#e6f4ea;border:1px solid #92c9a3;"><b>Geometria</b><br><small>vértices</small></div>
  <span>→</span>
  <div style="padding:8px 12px;border-radius:6px;background:#fef3e0;border:1px solid #e9bf75;"><b>Rasterização</b><br><small>triângulos → pixels</small></div>
  <span>→</span>
  <div style="padding:8px 12px;border-radius:6px;background:#fce8e6;border:1px solid #e79c94;"><b>Proc. de pixel</b><br><small>cor final</small></div>
</div>

1. **Aplicação:** roda na CPU e é onde o programador tem controle total: entrada do usuário, detecção de colisão, animação, física. No fim, ela envia as primitivas (vértices, arestas, triângulos) para a GPU.
2. **Geometria:** decide *o que* vai aparecer, *como* e *onde* na tela.
3. **Rasterização:** recebe três vértices (um triângulo) e descobre quais pixels ficam dentro dele.
4. **Processamento de pixel:** calcula a cor de cada pixel, testa a profundidade e combina o resultado com o que já estava desenhado.

## A jornada de um vértice

O estágio de geometria foi o que mais me chamou atenção, porque cada vértice passa por vários "espaços" de coordenadas até chegar à tela:

| Etapa | O que acontece |
|---|---|
| **Transformação de modelo** | O objeto sai do seu espaço local (ex.: uma esfera de raio 1 centrada em (0,0,0)) e, com translação, rotação e escala, vai para sua posição no **espaço do mundo**. |
| **Transformação de visão** | Tudo é reposicionado para que a câmera fique na origem, olhando para o **eixo Z negativo** (X para a direita, Y para cima). É o *espaço da câmera*. |
| **Vertex shading** | Calcula o efeito das luzes e dos materiais em cada vértice. |
| **Projeção** | Perspectiva (objetos distantes ficam menores, como na vida real) ou ortográfica (sem essa deformação, útil em CAD e jogos isométricos). |
| **Corte (clipping)** | Depois da projeção, o volume de visão vira um cubo unitário. O que está totalmente fora é descartado, e o que está parcialmente dentro é cortado nas bordas. |
| **Mapeamento na tela** | O cubo é escalado para o tamanho da janela, gerando as coordenadas de tela, que ainda guardam o Z (profundidade). |

Uma coisa que eu não tinha percebido antes é que a câmera virtual é definida por poucos parâmetros: **posição, direção, vetor *up*, campo de visão (FOV) e os planos *near* e *far***. Tudo que fica dentro desse volume em forma de pirâmide cortada (o *view frustum*) pode aparecer na imagem. O resto nem chega a ser desenhado.

## Rasterização e o problema da profundidade

Na rasterização, a GPU varre os pixels que caem dentro de cada triângulo e gera um **fragmento** para cada um. As propriedades do fragmento (cor, profundidade, coordenadas de textura) são **interpoladas** a partir dos três vértices.

Mas surge um problema: se dois triângulos cobrem o mesmo pixel, qual deles aparece? Desenhar "do mais longe para o mais perto" não resolve tudo, principalmente quando dois triângulos **se atravessam**. A solução usada pelas GPUs é o **Z-buffer**:

- para cada pixel, guardamos a profundidade (Z) do que já foi desenhado;
- ao desenhar um novo fragmento, comparamos o Z dele com o do buffer;
- se o novo for **menor** (mais perto da câmera), ele substitui a cor e o Z. Senão, é descartado.

Para entender melhor, montei a demonstração abaixo. Ela é um rasterizador de verdade, que testa pixel por pixel quem está dentro de cada triângulo e interpola o Z. Os dois triângulos se cruzam no espaço: o vermelho fica mais perto da câmera à esquerda, e o azul fica mais perto à direita.

<div id="zdemo" style="margin:1.5em 0;padding:14px;border:1px solid #ddd;border-radius:8px;background:#fafafa;">
  <canvas id="zcanvas" width="640" height="400" style="width:100%;height:auto;image-rendering:pixelated;border-radius:4px;background:#111;"></canvas>
  <div style="display:flex;flex-wrap:wrap;gap:8px;margin-top:10px;font-size:14px;">
    <button type="button" id="zbtn-z" style="padding:6px 12px;cursor:pointer;">Z-buffer: LIGADO</button>
    <button type="button" id="zbtn-ordem" style="padding:6px 12px;cursor:pointer;">Ordem: vermelho → azul</button>
    <button type="button" id="zbtn-vista" style="padding:6px 12px;cursor:pointer;">Mostrar: cor</button>
  </div>
  <p id="zinfo" style="margin:10px 0 0;font-size:14px;color:#555;"></p>
</div>

<script>
(function () {
  var W = 64, H = 40;
  var canvas = document.getElementById("zcanvas");
  if (!canvas) return;
  var ctx = canvas.getContext("2d");
  var S = canvas.width / W;
  // vértices: [x, y, z]  (z menor = mais perto da câmera)
  var vermelho = { cor: [230, 60, 60], v: [[3, 6, 0.10], [61, 10, 0.90], [26, 38, 0.50]] };
  var azul = { cor: [60, 120, 235], v: [[3, 34, 0.90], [61, 30, 0.10], [36, 2, 0.50]] };
  var usarZ = true, vermelhoPrimeiro = true, verProfundidade = false;

  function aresta(a, b, x, y) { return (b[0] - a[0]) * (y - a[1]) - (b[1] - a[1]) * (x - a[0]); }

  function rasterizar(tri, zbuf, cbuf) {
    var a = tri.v[0], b = tri.v[1], c = tri.v[2];
    var area = aresta(a, b, c[0], c[1]);
    for (var y = 0; y < H; y++) {
      for (var x = 0; x < W; x++) {
        var px = x + 0.5, py = y + 0.5;
        var w0 = aresta(b, c, px, py) / area;
        var w1 = aresta(c, a, px, py) / area;
        var w2 = aresta(a, b, px, py) / area;
        if (w0 < 0 || w1 < 0 || w2 < 0) continue;      // pixel fora do triângulo
        var z = w0 * a[2] + w1 * b[2] + w2 * c[2];      // Z interpolado
        var i = y * W + x;
        if (!usarZ || z < zbuf[i]) { zbuf[i] = z; cbuf[i] = tri.cor; }
      }
    }
  }

  function desenhar() {
    var zbuf = new Float32Array(W * H).fill(1), cbuf = new Array(W * H).fill(null);
    var ordem = vermelhoPrimeiro ? [vermelho, azul] : [azul, vermelho];
    ordem.forEach(function (t) { rasterizar(t, zbuf, cbuf); });

    var img = ctx.createImageData(W, H);
    for (var i = 0; i < W * H; i++) {
      var r, g, b;
      if (verProfundidade) {
        r = g = b = Math.round(255 * (1 - zbuf[i]));      // perto = claro, longe = escuro
        if (!cbuf[i]) r = g = b = 17;
      } else if (cbuf[i]) {
        var luz = 1 - 0.45 * zbuf[i];
        r = cbuf[i][0] * luz; g = cbuf[i][1] * luz; b = cbuf[i][2] * luz;
      } else { r = g = b = 17; }
      img.data[i * 4] = r; img.data[i * 4 + 1] = g; img.data[i * 4 + 2] = b; img.data[i * 4 + 3] = 255;
    }
    var tmp = document.createElement("canvas"); tmp.width = W; tmp.height = H;
    tmp.getContext("2d").putImageData(img, 0, 0);
    ctx.imageSmoothingEnabled = false;
    ctx.drawImage(tmp, 0, 0, canvas.width, canvas.height);

    ctx.strokeStyle = "rgba(255,255,255,0.06)";             // grade de pixels
    for (var x = 0; x <= W; x++) { ctx.beginPath(); ctx.moveTo(x * S, 0); ctx.lineTo(x * S, canvas.height); ctx.stroke(); }
    for (var y = 0; y <= H; y++) { ctx.beginPath(); ctx.moveTo(0, y * S); ctx.lineTo(canvas.width, y * S); ctx.stroke(); }

    document.getElementById("zbtn-z").textContent = "Z-buffer: " + (usarZ ? "LIGADO" : "DESLIGADO");
    document.getElementById("zbtn-ordem").textContent = "Ordem: " + (vermelhoPrimeiro ? "vermelho → azul" : "azul → vermelho");
    document.getElementById("zbtn-vista").textContent = "Mostrar: " + (verProfundidade ? "profundidade (Z)" : "cor");
    document.getElementById("zinfo").textContent = usarZ
      ? "Com o Z-buffer, cada pixel mostra o triângulo mais próximo, não importa a ordem. Repare na linha onde eles se cruzam."
      : "Sem o Z-buffer, o último triângulo desenhado cobre o outro inteiro, e nenhuma ordem fica correta, porque eles se atravessam.";
  }

  document.getElementById("zbtn-z").onclick = function () { usarZ = !usarZ; desenhar(); };
  document.getElementById("zbtn-ordem").onclick = function () { vermelhoPrimeiro = !vermelhoPrimeiro; desenhar(); };
  document.getElementById("zbtn-vista").onclick = function () { verProfundidade = !verProfundidade; desenhar(); };
  desenhar();
})();
</script>

O que mais me impressionou é que, com os triângulos se cruzando, **não existe ordem de desenho correta** sem o Z-buffer. A decisão precisa ser feita pixel a pixel, e é exatamente isso que a GPU faz no estágio de combinação (*merging*). No modo "profundidade" dá para ver o próprio conteúdo do Z-buffer: quanto mais claro, mais perto da câmera.

Além do Z-buffer, o *frame buffer* reúne outros buffers: o **canal alfa** (transparência), o **stencil buffer** (máscaras e espelhos) e o **double buffering**. Neste último, a cena é desenhada no *back buffer*, fora da tela, e só é trocada com o *front buffer* quando está pronta. É por isso que não vemos a imagem sendo montada aos pedaços.

## O gargalo manda em tudo

Outro conceito importante da aula é o de **gargalo (bottleneck)**: como os estágios trabalham em sequência, a velocidade da pipeline inteira é definida pelo **estágio mais lento**. Por exemplo:

- aplicação: 4 ms
- geometria: 6 ms
- processamento de pixel: 20 ms

Mesmo que eu deixe a geometria duas vezes mais rápida, o jogo continua limitado a 1000 / 20 = **50 FPS**, porque o processamento de pixel continua levando 20 ms. Otimizar um estágio que não é o gargalo é esforço desperdiçado. Por isso, antes de otimizar, é preciso **descobrir onde está o gargalo**. Isso vale para gráficos e também para praticamente qualquer sistema em computação.

## Conclusão

Antes da aula, eu via a renderização como uma "caixa-preta" que transformava modelos 3D em imagens. Agora consigo enxergar o caminho: o vértice é transformado entre espaços de coordenadas, projetado, cortado, mapeado para a tela, vira fragmentos na rasterização e, por fim, passa pelo teste de profundidade antes de virar um pixel colorido. E tudo isso acontece milhões de vezes por segundo.

---

*Referências: slides da aula "Pipeline Gráfica" (Prof. André Kishimoto, 2026), baseados em AKENINE-MÖLLER, T. et al. **Real-Time Rendering**, 4ª ed. CRC Press, 2018.*
