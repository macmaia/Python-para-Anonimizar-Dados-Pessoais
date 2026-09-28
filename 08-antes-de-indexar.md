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
    description: "Por que mascarar dado pessoal antes de indexar em RAG e diferente de mascarar antes de uma chamada: o chunk parte o CPF no meio, a pergunta tambem carrega PII e o indice persiste."
---

# 8. Antes de indexar, não só antes de perguntar

O capítulo 5 tratou do texto que vai para o modelo em uma chamada. Hoje a maior parte do esforço de
anonimização não está ali, está na ingestão para busca com recuperação, o RAG. O documento é partido em
pedaços, cada pedaço vira vetor, e o índice fica.

A diferença não é de grau. Mascarar antes de uma chamada é decisão sobre um texto que passa. Mascarar antes de
indexar é decisão sobre um texto que **permanece**, num lugar de onde é difícil tirar.

## Por que o chunk quebra a detecção?

Porque o dígito verificador precisa dos onze dígitos juntos, e o corte não sabe disso.

```{code-cell}
import tarja

documento = "Contrato firmado com o titular, CPF 529.982.247-25, em 12/03/2024."
tamanho = 40

pedacos = [documento[i:i + tamanho] for i in range(0, len(documento), tamanho)]
for p in pedacos:
    print(f"{p!r:45} -> {[m.entity for m in tarja.find(p)]}")

print()
print("documento inteiro ->", [m.entity for m in tarja.find(documento)])
```

O documento inteiro tem um CPF. Nenhum dos pedaços tem. O corte caiu dentro do número, e o que sobrou de cada
lado não é CPF nem por formato nem por aritmética.

Isso é pior do que um falso negativo comum, porque é **silencioso e sistemático**. Não depende de texto
difícil, de OCR nem de abreviação. Depende do tamanho da janela, que é decisão de quem montou o pipeline de
indexação e que quase nunca é revista pensando em dado pessoal.

## Onde aplicar, então?

Antes de partir, no documento inteiro.

```{code-cell}
mascarado = tarja.mask(documento)
print(mascarado)
print()
for p in [mascarado[i:i + tamanho] for i in range(0, len(mascarado), tamanho)]:
    print(f"  {p!r}")
```

Nenhum pedaço carrega dado pessoal, e na ingestão é isso que importa.

Repare no efeito colateral, visível na saída: o corte agora parte o **token**, e não o número. Um pedaço fica com
`<BR_` e o outro com `CPF>`. Nada vazou, porque o token não é o valor, mas um token partido não é reversível pelo
`reveal` do capítulo 5, e é mais uma razão para o `reveal` não pertencer a este desenho.

A regra que vale decorar: **mascarar precede partir**, e não o contrário.

Se o pipeline que você herdou já parte antes de qualquer tratamento, o paliativo é janela com sobreposição,
para que todo identificador apareça inteiro em pelo menos um pedaço:

```{code-cell}
sobreposicao = 20
for p in [documento[i:i + tamanho + sobreposicao] for i in range(0, len(documento), tamanho)]:
    print(f"{p!r:66} -> {[m.entity for m in tarja.find(p)]}")
```

Funciona, e é remendo. A sobreposição precisa ser maior que o maior identificador esperado, ninguém verifica
isso, e o custo de armazenamento sobe. Prefira mascarar antes.

## A pergunta também carrega dado pessoal

Quem pergunta "o que consta do processo do CPF 529.982.247-25" acabou de enviar um CPF para o mesmo modelo, e
provavelmente para o mesmo log. São **dois pontos de aplicação**, não um: a ingestão e a consulta.

```{code-cell}
pergunta = "o que consta do processo do CPF 529.982.247-25?"
print(tarja.mask(pergunta))
```

O ponto da consulta é o mais esquecido dos dois, porque o time que cuidou da ingestão considerou o problema
resolvido. E é o mais frequente, porque a ingestão acontece uma vez e a pergunta acontece todo dia.

## Por que o pseudônimo precisa ser estável aqui?

Porque num índice a utilidade da busca depende de o mesmo identificador virar o mesmo token em documentos
diferentes. Se cada documento receber um token distinto para o mesmo CPF, você não recupera mais tudo sobre a
mesma pessoa, que muitas vezes é o objetivo da consulta. É o `pseudonym_stable` do capítulo 4.

```{code-cell}
import secrets

chave = secrets.token_hex(32)
for d in ["Autuação lavrada contra o titular do CPF 529.982.247-25.",
          "Recurso apresentado pelo titular do CPF 529.982.247-25."]:
    print(tarja.mask(d, strategy="pseudonym_stable", salt=chave))
```

O mesmo token nos dois. É isso que preserva a capacidade de juntar, sem guardar o valor.

## O que o tarja não resolve nesse desenho

**O cofre não atravessa o índice.** O `reveal()` do capítulo 4 depende de um cofre local ao processo, de uso
único e com validade de uma hora. Isso foi escolhido de propósito, para que o mapa entre token e valor não
virasse um banco de dado pessoal. Índice vetorial vive meses. Na prática: o `mask` serve para a ingestão, e o
`reveal` não serve para devolver o valor original numa consulta feita semanas depois. Se o seu caso exige isso,
você precisa de um guarda-chaves próprio, com o ciclo de vida e a auditoria que ele implica, e isso é decisão
de arquitetura e não chamada de função.

**A ordem é irreversível.** Mascare e depois embede, nunca o contrário, porque não existe desmascarar um vetor.
Se o índice foi construído com texto cru, o conserto é reindexar.

**Exclusão é problema aberto.** Pedido de eliminação sob a LGPD contra um índice vetorial não é questão de
detecção, é de engenharia de armazenamento. Localizar todos os pedaços de uma pessoa exige que você tenha
guardado essa ligação, que é justamente o que a pseudonimização evita guardar. Não há resposta limpa aqui, e um
livro que fingisse haver seria pior que este.

## O que fica

O RAG não é caso especial de "mandar texto para o modelo". É o caso em que o texto fica, o corte destrói a
aritmética que reconheceria o dado, e a pergunta é um segundo ponto de vazamento que quase ninguém protege.
