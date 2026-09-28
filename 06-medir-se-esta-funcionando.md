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
    description: "Como medir a sua propria taxa de erro em deteccao de dado pessoal, com amostragem, intervalo de confianca e precisao e recall calculados no seu corpus."
---

# 6. Medir se está funcionando

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

Até aqui o livro mostrou o que fazer. Este capítulo é sobre saber se funcionou no **seu** texto, que é diferente
de funcionar no texto de quem escreveu a ferramenta.

## Os dois erros não custam a mesma coisa

Um detector erra de duas formas. Ele aponta o que não é, e deixa passar o que é.

**Falso positivo** é o número de protocolo marcado como CPF. O custo imediato é destruir informação que a
operação usa, se você mascarar. O custo maior é o do capítulo 1: quando a lista tem ruído demais, ninguém
confia nela, inclusive na parte certa.

**Falso negativo** é o CPF que continua no texto depois de você ter dito que tratou. O custo é o dado pessoal
exposto, e o agravante é que você não sabe que ele está lá. Um relatório de falso positivo é irritante. Um
falso negativo é o que aparece no incidente.

Qual dos dois apertar depende do que você está fazendo. Mascarando antes de mandar para fora, falso negativo é
pior, e vale aceitar ruído. Produzindo um inventário para o encarregado decidir prioridade, falso positivo é
pior, porque um mapa errado leva a decisão errada.

Não existe configuração que minimize os dois. Existe escolher qual deles você prefere cometer.

## O que são precisão e revocação aqui?

**Precisão** responde: do que eu apontei, quanto estava certo. **Revocação** responde: do que existia, quanto
eu achei.

```{code-cell}
def avaliar(gold: set, pred: set) -> dict:
    vp = len(gold & pred)
    fp = len(pred - gold)
    fn = len(gold - pred)
    precisao = vp / (vp + fp) if vp + fp else 0.0
    revocacao = vp / (vp + fn) if vp + fn else 0.0
    f1 = 2 * precisao * revocacao / (precisao + revocacao) if precisao + revocacao else 0.0
    return {"VP": vp, "FP": fp, "FN": fn, "precisao": round(precisao, 3),
            "revocacao": round(revocacao, 3), "F1": round(f1, 3)}

# gold: o que você anotou à mão. pred: o que a ferramenta achou.
gold = {(13, 27, "BR_CPF"), (40, 58, "BR_CNS")}
pred = {(13, 27, "BR_CPF"), (70, 81, "BR_CPF")}
print(avaliar(gold, pred))
```

Repare que um achado conta como certo só se **o tipo e a posição** baterem. Acertar que há algo ali e errar o
tipo é erro, porque o tratamento depende do tipo.

## Como amostrar e anotar o seu próprio corpus

Precisão exige saber a resposta certa, e a resposta certa não vem de graça. Alguém precisa olhar e julgar.

O atalho que quase todo mundo usa é rodar a ferramenta no seu texto, contar quantos achados ela produziu, e
apresentar isso como resultado. Isso não mede nada: é a ferramenta se avaliando.

O procedimento honesto é curto e chato:

1. Rode a detecção no seu acervo e guarde **todos** os achados.
2. Sorteie uma amostra aleatória, com semente fixa para poder repetir.
3. Olhe um por um, com o texto em volta, e diga: é mesmo um identificador desse tipo?
4. Precisão é a proporção de sim, com intervalo de confiança.

Uma amostra de cem leva de uma a duas horas e é o único número que você pode defender. Duzentos achados
conferidos por amostragem valem mais que quarenta mil não conferidos.

Repare que isso mede **precisão** e não revocação. Revocação exige o caminho inverso: pegar documentos e
anotá-los por inteiro, do zero, sem olhar a saída da ferramenta. É bem mais caro, e é por isso que quase todo
relatório do mercado informa só precisão, sem dizer que está informando só metade.

## Por que reportar intervalo de confiança?

```{code-cell}
def wilson(acertos: int, total: int, z: float = 1.96) -> tuple[float, float]:
    """Intervalo de Wilson: honesto com amostra pequena e perto de 0 ou 1."""
    if not total:
        return (0.0, 0.0)
    p = acertos / total
    d = 1 + z * z / total
    centro = (p + z * z / (2 * total)) / d
    meio = z * ((p * (1 - p) / total + z * z / (4 * total * total)) ** 0.5) / d
    return (max(0.0, centro - meio), min(1.0, centro + meio))

for acertos, total in [(39, 39), (36, 39), (95, 100)]:
    lo, hi = wilson(acertos, total)
    print(f"{acertos}/{total}: precisão {acertos/total:.3f}  IC95% [{lo:.3f}, {hi:.3f}]")
```

Olhe a primeira linha. Trinta e nove acertos em trinta e nove julgamentos dá precisão 1,000, e a fórmula
clássica de proporção daria um intervalo de 1,000 a 1,000: certeza absoluta a partir de trinta e nove
observações, o que é obviamente falso. Wilson diz de 0,910 a 1,000, que é o que os dados sustentam.

É por isso que ele é usado aqui. Perto de zero ou de um, e com amostra pequena, a aproximação normal mente na
direção mais conveniente.

Quando alguém apresentar precisão sem intervalo, a pergunta é: sobre quantos julgamentos.

## Suspeito: formato certo, dígito errado

Existe uma terceira categoria, e ela não é falso positivo nem falso negativo.

```{code-cell}
import tarja

for m in tarja.find("cpf 529.982.247-24", report_invalid=True):
    print(m.entity, "dígito válido:", m.valid_dv, "score:", m.score)
```

Um número com formato de CPF cujo dígito não fecha. Não é reportado como achado, e por isso não entra na
prevalência. Mas ele quase nunca é coincidência: é um CPF real com um dígito digitado errado, ou lido errado
por reconhecimento óptico.

Duas consequências. Volume alto de suspeitos indica problema na origem, e é diagnóstico útil. E **um suspeito
é quase dado pessoal**: reidentifica a mesma pessoa em nove de cada dez casos, porque um dígito trocado ainda
deixa dez candidatos e o contexto resolve.

Então suspeito não vai para log, não vai para mensagem de erro, não vai para o monitoramento. Conte, ou carregue
sem o valor:

```{code-cell}
achado = tarja.find("Paciente CPF 529.982.247-25")[0]
print(achado.to_dict(include_value=False))
```

E imprimir um achado é seguro, porque o valor não aparece:

```{code-cell}
print(tarja.find("Paciente CPF 529.982.247-25"))
```

Esse comportamento existe justamente porque a instrução "não logue" é fácil de escrever na documentação e
fácil de esquecer no código.

## O que medir de tempos em tempos

Uma medição é uma fotografia. O seu texto muda: sistema novo, formato novo, fornecedor novo, campo que passou
a ser preenchido de outro jeito. Uma precisão medida em março não diz nada sobre outubro.

Repita a amostragem quando mudar a fonte dos dados, quando atualizar a ferramenta, e de resto uma vez por
semestre. Guarde a semente e o tamanho da amostra junto com o número, senão você não consegue comparar com a
medição anterior.

## O que fica

Falso positivo e falso negativo custam coisas diferentes, e você escolhe qual cometer. Precisão só existe se
alguém olhou e julgou uma amostra. Intervalo de confiança não é enfeite estatístico, é o que impede de afirmar
certeza a partir de trinta e nove observações. E suspeito é quase dado pessoal, não é saída de depuração.

O último capítulo é sobre o que nada disso faz.
