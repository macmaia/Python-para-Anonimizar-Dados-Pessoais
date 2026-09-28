---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# Python para anonimizar dados pessoais

Você recebeu a tarefa de tirar dado pessoal de um sistema que já está rodando. Talvez tenha vindo do
encarregado, talvez de uma auditoria, talvez de alguém que leu a LGPD e ficou preocupado. A tarefa chegou
como uma frase curta, e a frase esconde um problema difícil.

Este livro é sobre esse problema. Ele ensina a encontrar identificadores pessoais em texto em português, a
decidir o que fazer com cada um, e a medir se funcionou. Tudo com código que roda.

## Para quem é

Para quem programa em Python e tem texto em português na mão. Log de aplicação, chamado de suporte, contrato,
prontuário, e-mail, planilha exportada, prompt indo para um modelo de linguagem.

Não é um livro de direito. A LGPD aparece onde ela muda o que o código precisa fazer, e só aí.

## O que você vai conseguir fazer no fim

Explicar por que procurar CPF com uma expressão regular produz relatório inútil, e o que usar no lugar.
Escrever a validação de um identificador do zero, entendendo a regra. Decidir entre apagar, trocar por rótulo
e trocar por token reversível, sabendo o que cada escolha permite e impede depois. Mandar texto para um modelo
de linguagem sem mandar o dado pessoal junto. E medir a sua própria taxa de erro, nos dois sentidos.

## Como rodar

Cada capítulo tem código executável. O que está impresso nas páginas é a saída real, produzida quando o livro
foi construído, não texto copiado à mão.

Para acompanhar na sua máquina:

```bash
pip install tarja
```

A partir do capítulo 2 o livro usa o [tarja](https://macmaia.github.io/tarja/), uma biblioteca aberta sob
Apache 2.0, sem dependência de execução. O capítulo 2 escreve a regra do CPF à mão antes de mostrar a
biblioteca, de propósito: você precisa entender o mecanismo para decidir se confia nele.

## O que este livro não cobre

Nome de pessoa, endereço e dado clínico em prosa. Isso depende de reconhecimento de entidade nomeada, que é
outro problema, com outra literatura e outra taxa de erro. O capítulo 7 é inteiro sobre os limites, e ele
existe porque um livro que só mostra o que funciona não serve para trabalhar.

---

Escrito por [Maria Alice Maia](https://github.com/macmaia). Texto sob CC BY 4.0, código sob Apache 2.0.
