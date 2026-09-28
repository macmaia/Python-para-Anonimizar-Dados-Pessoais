---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
myst:
  html_meta:
    description: "Por que uma regex de formato para CPF gera falso positivo em massa, e quanto ruido ela aceita de fato, medido em Python."
---

# 1. Por que procurar por formato não funciona

```{code-cell}
:tags: [skip-execution]
# No Colab, rode esta célula primeiro. Na construção do livro ela é pulada, porque o tarja já está instalado.
%pip install -q tarja
```

Um CPF tem onze dígitos. A primeira solução que quase todo mundo escreve é uma expressão regular que procura
onze dígitos. Este capítulo mostra por que ela não serve, com números, e o que está faltando.

## Por que uma regex de CPF não basta?

```{code-cell}
import re

CPF_POR_FORMATO = re.compile(r"\b\d{3}\.?\d{3}\.?\d{3}-?\d{2}\b")

texto = """
Prezado cliente, seu chamado foi aberto sob o protocolo 20240315001.
Segue o CPF do titular para conferência: 529.982.247-25.
O número de processo interno é 18273645902 e a matrícula do atendente é 30912847561.
"""

for achado in CPF_POR_FORMATO.finditer(texto):
    print(achado.group())
```

Quatro achados. Um é CPF. Os outros três são um protocolo, um número de processo interno e uma matrícula de
funcionário, todos com onze dígitos porque onze dígitos é um tamanho banal.

Isso não é um caso construído para o livro. Texto administrativo brasileiro é cheio de número comprido:
protocolo, matrícula, número de processo, código de rastreamento, identificador interno de sistema. Três
quartos do que a expressão regular achou ali é ruído, e num acervo de verdade a proporção é pior, porque
documento administrativo tem muito mais protocolo que CPF.

## Quanto ruído a regex de formato aceita?

Quanto disso é ruído? Dá para estimar sem adivinhar. Vamos gerar sequências de onze dígitos ao acaso e contar
quantas a expressão regular aceita.

```{code-cell}
import random

rng = random.Random(2026)
sequencias = ["".join(rng.choice("0123456789") for _ in range(11)) for _ in range(100_000)]
aceitas = sum(1 for s in sequencias if CPF_POR_FORMATO.fullmatch(s))
print(f"{aceitas} de {len(sequencias)} aceitas pelo formato, ou seja {100*aceitas/len(sequencias):.0f}%")
```

Cem por cento. A expressão regular não rejeita nada que tenha o tamanho certo, porque ela só sabe contar
dígitos. Todo número de onze dígitos do seu acervo vira um suposto CPF.

## Por que isso é pior do que parece

Um relatório com falso positivo demais não é um relatório ruim, é um relatório que ninguém usa.

Imagine entregar ao time de engenharia uma lista de quarenta mil supostos CPFs, dos quais talvez dois mil
sejam CPFs. A primeira reação vai ser conferir uma amostra, ver que a maioria é protocolo, e parar de confiar
na lista inteira, incluindo os dois mil que estavam certos. O custo do falso positivo não é o tempo de
conferir, é a perda de credibilidade do instrumento.

E tem o efeito contrário, mais perigoso. Se você mascarar tudo o que essa expressão encontra, vai destruir
número de protocolo, matrícula e processo, que a operação precisa. Alguém vai reclamar, e a resposta fácil
vai ser afrouxar a regra, o que traz de volta o dado pessoal que você queria tirar.

## O que falta é o dígito verificador

Um CPF não é qualquer sequência de onze dígitos. Os dois últimos dígitos são **calculados** a partir dos nove
primeiros, por uma regra pública e determinística. Um número em que essa conta não fecha não é um CPF.

Esse é o ponto de virada deste livro. A informação que separa um CPF de um protocolo não está no formato, está
na **regra de formação**. E identificador brasileiro tem regra de formação em abundância: CPF, CNPJ, cartão do
SUS, número de processo judicial, título de eleitor, PIS, RENAVAM, CNH. Quase todos carregam dígito
verificador.

Dá para ver o efeito sem ainda saber a regra. Vamos usar a verificação pronta e repetir a contagem:

```{code-cell}
import tarja

validos = sum(1 for s in sequencias if tarja.validate("BR_CPF", s))
print(f"{validos} de {len(sequencias)} passam na regra de formação, ou seja {100*validos/len(sequencias):.2f}%")
print(f"redução de ruído: {aceitas/max(validos,1):.0f} vezes")
```

De cem por cento para cerca de um por cento: quase cem vezes menos ruído, e sem nenhuma informação externa,
só aplicando uma regra pública.

Voltando ao texto do começo:

```{code-cell}
for achado in CPF_POR_FORMATO.finditer(texto):
    valor = achado.group()
    print(f"{valor:18} {'CPF' if tarja.validate('BR_CPF', valor) else 'nao e CPF'}")
```

Os três números que não eram CPF somem. O CPF fica.

## Quanto custa um falso positivo de CPF

Nada é de graça, e vale saber o preço antes de comemorar.

A regra de formação reduz falso positivo, e **não reduz falso negativo**. Um CPF que você não procurou continua
onde estava. Pior: um CPF digitado errado, com um dígito trocado, deixa de passar na conta e some do relatório,
mesmo sendo dado pessoal real de uma pessoa real. O capítulo 6 trata desses dois casos, porque eles custam
coisas diferentes e a maioria das ferramentas comerciais não distingue.

E dígito verificador válido **não** quer dizer que o número pertence a alguém. Quer dizer que ele é bem
formado. Um CPF gerado ao acaso que por sorte fecha a conta passa igual. Este livro nunca consulta base
externa e nunca resolve identificador em pessoa, e o capítulo 7 explica por que essa fronteira é deliberada.

## O que fica

Procurar dado pessoal por formato produz ruído na proporção de cem para um. A informação que falta está na
regra de formação, que é pública e verificável.

No próximo capítulo você escreve essa regra à mão, em quinze linhas, antes de usar qualquer biblioteca.
