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
    description: "O que fazer com CEP, placa e telefone, que nao tem digito verificador, e por que filtrar por score nao resolve o falso positivo que esta perto de uma palavra de contexto."
---

# 3. Quando o formato não basta e o contexto decide

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

Nem todo identificador tem dígito verificador. CEP, placa e telefone não têm: são apenas formatos. E alguns que
têm dígito verificador ainda assim colidem com números legítimos, porque o espaço de valores válidos é grande
o bastante para que o acaso produza colisão.

Nos dois casos, a única informação adicional disponível é o que está escrito em volta.

## Por que o CEP exige uma palavra de contexto?

```{code-cell}
import tarja

print(tarja.find("04543-907"))
```

Vazio. Cinco dígitos, traço, três dígitos é um formato que qualquer coisa pode ter: faixa de numeração, código
de produto, intervalo. Sozinho, não é informação suficiente.

```{code-cell}
for m in tarja.find("CEP 04543-907"):
    print(m.entity, "score", m.score, "contexto", m.has_context)
```

A palavra muda tudo. É a mesma sequência de dígitos, e o que apareceu foi o contexto.

Essa é a diferença entre **exigir contexto** e **usar contexto para pontuar**. O CEP exige: sem a palavra, não
é reportado. A placa não exige, ela aparece com score mais baixo e sobe quando há contexto.

```{code-cell}
for texto in ["ABC1D23", "veiculo de placa ABC1D23"]:
    achados = tarja.find(texto)
    score = achados[0].score if achados else None
    print(f"{texto!r:28} -> score {score}")
```

## Como a janela de contexto compara o texto?

A janela de contexto olha alguns caracteres antes e depois do achado, e procura ali as palavras da entidade. A
comparação acontece sobre texto em minúscula e sem acento, e por isso todas estas funcionam:

```{code-cell}
for texto in ["CEP 04543-907", "cep 04543-907", "Cep: 04543-907", "CÉP 04543-907"]:
    print(f"{texto!r:22} -> {len(tarja.find(texto))} achado(s)")
```

Isso importa mais do que parece. Texto real tem caixa inconsistente, acento errado, abreviação e pontuação
colada. Uma comparação literal perderia a maior parte dos casos, e quem estivesse medindo concluiria que a
detecção é ruim quando na verdade o pré-processamento é que estava.

## Por que filtrar por score não separa título de ano?

Agora o exemplo que dá nome a este capítulo.

Um título de eleitor tem doze dígitos e dígito verificador. Veja o que acontece com uma sequência de anos:

```{code-cell}
for m in tarja.find("Anos de referencia: 1999 2001 2003 2005", report_invalid=False):
    print(m.entity, "score", m.score, "contexto", m.has_context)
```

`1999 2001 2003` **passa no dígito verificador**. Não é bug do detector nem do algoritmo: o espaço de títulos
válidos é grande, e uma sequência de quatro anos consecutivos cai dentro dele por acaso.

O dígito verificador resolveu o problema do capítulo 1 e não resolve este. Por isso o título usa contexto.

```{code-cell}
titulo = "7097 1071 1422"   # sintético, dígito verificador válido

for texto in [titulo, f"Inscricao: {titulo}", f"titulo de eleitor {titulo}"]:
    m = tarja.find(texto)[0]
    print(f"{texto!r:40} score {m.score}  contexto {m.has_context}")
```

Com contexto, 0,95. Sem contexto, 0,8. Então parece que basta filtrar por score, e é isso que quase todo mundo
faz:

```{code-cell}
texto = f"Eleitora, inscricao {titulo}. Anos de referencia: 1999 2001 2003 2005."

for m in tarja.find(texto, min_score=0.9):
    print(f"{m.entity} score {m.score} contexto {m.has_context} -> {texto[m.start:m.end]!r}")
```

**Os dois passam.** A sequência de anos também recebeu 0,95, porque a palavra "inscrição" está a menos de
sessenta caracteres dela. A janela de contexto mede proximidade, não relação gramatical, e não tem como saber
que aquela palavra se refere ao outro número.

Separe os dois no texto e o comportamento muda:

```{code-cell}
longe = (f"Eleitora, inscricao {titulo}. "
         + "Texto intermediario sem nada de especial. " * 3
         + "Anos de referencia: 1999 2001 2003 2005.")

for m in tarja.find(longe, min_score=0.0):
    print(f"score {m.score} contexto {m.has_context} -> {longe[m.start:m.end]!r}")
```

Agora sim: 0,95 para o título, 0,8 para os anos, e o filtro funciona.

## O limiar é propriedade do seu texto, não do método

O limiar de score separa verdadeiro de falso **quando o falso positivo está longe de qualquer palavra de
contexto**. Isso é uma propriedade do seu texto, não do método.

Num acervo onde cada identificador aparece num campo rotulado, com o rótulo colado nele, a separação funciona
muito bem. Num acervo de prosa corrida, onde a palavra "inscrição" pode estar em qualquer lugar do parágrafo,
ela funciona pior. Você não descobre em qual dos dois está lendo a documentação. Descobre medindo, e o capítulo
6 mostra como.

Este é o tipo de limitação que ferramenta comercial não conta. Ela publica a acurácia num conjunto de teste e
deixa você supor que ela se transfere para o seu texto. Não se transfere, e supor isso é como a maioria dos
projetos de detecção de dado pessoal termina com um relatório em que ninguém confia.

## Escolher a janela é escolher um erro

Janela maior pega o rótulo que está longe, e por isso reduz falso negativo. Janela maior também alcança
palavras que não têm relação com o número, e por isso aumenta falso positivo. Não existe valor certo, existe
qual dos dois erros custa mais no seu caso.

A janela do título é de sessenta caracteres. Se o seu texto tem rótulo sempre colado, um valor menor seria mais
preciso. Se tem formulário com muito espaço, um valor maior pegaria mais.

## O que fica

Sem dígito verificador, o contexto é tudo o que existe, e ele mede proximidade e não sentido. Score alto quer
dizer "há uma palavra de contexto perto", e perto não quer dizer relacionado.

O próximo capítulo muda de assunto: achado o identificador, o que fazer com ele. E as três opções não são graus
da mesma coisa.
