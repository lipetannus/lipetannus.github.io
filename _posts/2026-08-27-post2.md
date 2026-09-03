---
layout: post
title: "Como uma imagem vira números: Amostragem e Quantização"
date: 2026-08-27
---

Para o computador, uma imagem não passa de uma matriz bidimensional onde cada elemento é chamado de pixel[cite: 2]. Mas como capturamos a luz contínua do mundo real e a transformamos nessa matriz? Esse processo de discretização envolve duas etapas principais[cite: 2]:

1. **Amostragem (Sampling):** É o processo de digitalização dos valores de coordenada, o que define a resolução da imagem[cite: 2].
2. **Quantização (Quantization):** É a digitalização dos valores de amplitude, definindo os níveis de intensidade (ou níveis de cinza) que a imagem terá[cite: 2]. 

Por exemplo, em uma imagem de 8 bits, temos $2^8 = 256$ valores discretos possíveis de intensidade, variando do preto (valor 0) ao branco (valor 255)[cite: 2]. Entender esses dois passos é o ponto de partida para qualquer processamento digital de imagens.
