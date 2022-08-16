---
title: "Suporte ao Grego usando padoc"
date: 2022-08-14T18:21:12-03:00
slug: ""
description: "Como configurar uma font específica para um idioma usando pandoc."
keywords: []
draft: true
tags: ["pandoc", "grego"]
math: false
toc: false
---

Nessa semana, estava escrevendo algumas notas para um projeto que envolve
algumas citações em grego. Ando utilizando o `pandoc` por praticidade. A
vantagem principal dele é a possibilidade de escrever usando `markdown` e
compilar um PDF sem muitas complicações. Dessa forma, se posso compartilhar o
que escrevi com alguém de uma forma sã, imprimir para levar a alguma lugar, etc.
É tão simples quanto rodar:

```shell
pandoc file.md -o file.pdf
```

O que acontece: se você quiser utilizar uma

+ fonte principal precisa ter suporte ao grego (builtin .. não tem)
+ se sua fonte principal não tiver caracteres gregos (meu caso), é preciso
    configurar algo que possa fazer esse switch
+ polyglossia:
    + configurações para uma língua específica
        + como fazer
    + macro para indicar a troca de idioma em apenas um trecho

---

+ Artigo sobre setup do vim para iterações rápidas de compilação de documento
    + vim dispatch
    + au let dispatch ...
    + zathura open ...
    + talvez trablhar xdg-open
