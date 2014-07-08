---
layout: post
title: "(PT-BR) Palestra sobre Celluloid no RubyOnRio"
date: 2014-03-21 16:10
comments: true
categories: [ruby, celluloid, actors, akka, erlang]
---

No dia 15 de março de 2014, aproveitei o encontro do [RubyOnRio][rubyonrio] para falar sobre [Celluloid][celluloid], uma implementação do Actor Model para Ruby. Como programação reativa, concorrência, paralelismo e quetais têm ocupado minha mente nos últimos meses, achei por bem conversar com o pessoal sobre como isso é relevante para o futuro do desenvolvimento de software e para o Ruby em si, constantemente sob ameaça (se Hacker News for parâmetro) de ser soterrado por uma tecnologia mais antenada com os novos tempos.

<script async class="speakerdeck-embed" data-id="7acca2a0935901315c4a3abe98d15494" data-ratio="1.34031413612565" src="//speakerdeck.com/assets/embed.js"></script>

A ideia da minha apresentação foi dar uma pincelada nos problemas clássicos de threading, explicar _en passant_ as ideias do Actor Model, e por fim encerrar na aplicação disso dentro do Celluloid. Claro que há informações vitais que ficaram de fora, que a superficialidade pode ser criticada, que o palestrante é meio capenga, mas espero que no cômputo geral o resultado tenha agradado.

<iframe width="560" height="315" src="//www.youtube.com/embed/0t0BlDdWQQY" frameborder="0" allowfullscreen></iframe>
<iframe width="560" height="315" src="//www.youtube.com/embed/_y6KbkqklkQ" frameborder="0" allowfullscreen></iframe>

Por fim apresentei um projetinho que fiz especialmente para o encontro, o [Balladina][balladina]. Foi divertido fazê-lo, e aprendi bastante coisa sobre o Celluloid no processo. Pude também contrastar algumas coisas com a parca experiência que tive com o Akka no [curso do Coursera](/blog/2014/01/12/reactive/), e a maturidade do Celluloid relativa à do Akka me deixou esperançoso de um futuro bacana no Ruby. Vamos torcer pelo melhor.

[rubyonrio]: http://rubyonrio.org
[celluloid]: https://github.com/celluloid/celluloid
[balladina]: https://github.com/dodecaphonic/balladina-ruby
