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

# 2. O dígito verificador, escrito do zero

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

O capítulo anterior terminou dizendo que a informação que falta está na regra de formação. Agora você escreve
essa regra. São quinze linhas, e escrevê-las muda a forma como você julga qualquer biblioteca que faça isso
por você depois.

## A conta

Os nove primeiros dígitos de um CPF são o número. Os dois últimos são conferência, calculados assim: multiplique
cada dígito por um peso que decresce, some, tire o resto por onze, e transforme o resto no dígito.

```{code-cell}
def cpf_digito(base: str) -> str:
    """Calcula um dígito verificador de CPF a partir dos dígitos anteriores."""
    n = len(base) + 1
    soma = sum(int(d) * (n - i) for i, d in enumerate(base))
    resto = (soma * 10) % 11
    return "0" if resto == 10 else str(resto)

print(cpf_digito("529982247"))    # primeiro dígito
print(cpf_digito("5299822472"))   # segundo, já contando o primeiro
```

Para os nove primeiros dígitos, `n` vale 10, e os pesos vão de 10 até 2. Para os dez primeiros, `n` vale 11, e
os pesos vão de 11 até 2. É a mesma função chamada duas vezes, e a segunda chamada inclui o dígito que a
primeira produziu.

O `resto == 10` existe porque o resto da divisão por onze pode dar dez, que não cabe num dígito. A convenção é
usar zero.

## O validador

```{code-cell}
def cpf_valido(valor: str) -> bool:
    d = "".join(c for c in valor if c.isdigit())
    if len(d) != 11:
        return False
    if len(set(d)) == 1:
        return False
    return cpf_digito(d[:9]) == d[9] and cpf_digito(d[:10]) == d[10]
```

Quinze linhas no total, contando a função anterior. Nenhuma dependência.

```{code-cell}
casos = [
    ("529.982.247-25", True),
    ("529.982.247-24", False),   # um dígito trocado
    ("52998224725",    True),    # sem pontuação
    ("123.456.789-09", True),
    ("111.111.111-11", False),
    ("000.000.000-00", False),
]
for valor, esperado in casos:
    obtido = cpf_valido(valor)
    print(f"{valor:18} {str(obtido):5} {'ok' if obtido == esperado else 'ERRO'}")
```

## A linha que parece sobrando

Olhe de novo para esta:

```python
if len(set(d)) == 1:
    return False
```

Ela rejeita `111.111.111-11` e os outros nove números de dígito repetido. Parece defensividade inútil, e não é.
Confira:

```{code-cell}
print("primeiro dígito calculado para 111111111 :", cpf_digito("111111111"))
print("segundo dígito calculado para 1111111111:", cpf_digito("1111111111"))
```

**A conta fecha.** `111.111.111-11` satisfaz o módulo 11 perfeitamente. Sem aquela linha, ele passa como CPF
válido, e o mesmo vale para `222.222.222-22` e os demais.

Isso não é curiosidade matemática, é a diferença entre um validador correto e um quase correto. Sequências
repetidas são o valor de preenchimento mais comum que existe: formulário com campo obrigatório, teste que
alguém deixou no banco, importação que preencheu o vazio. Um validador sem essa regra vai marcar cada um deles
como CPF, e você vai passar uma tarde investigando por que há dez mil CPFs idênticos no relatório.

Essa regra não está no algoritmo. Está na prática, e você só descobre quando bate nela.

## As outras regras, em uma frase cada

A mesma ideia reaparece com variações, e é aqui que reimplementar deixa de ser razoável.

**CNPJ** é módulo 11 com pesos diferentes, sobre doze dígitos em vez de nove. E desde a mudança recente ele
aceita letras: o cálculo passa a usar o valor ASCII do caractere menos 48, o que faz `0` continuar valendo 0 e
`A` valer 17. Implementação antiga quebra em silêncio nesse caso, porque ela converte para inteiro e levanta
exceção, ou pior, descarta a letra.

**NIS, título de eleitor, RENAVAM e CNH** são módulo 11 com pesos próprios e convenções próprias para o resto.
O título ainda tem uma regra especial para as unidades federativas 01 e 02.

**Processo judicial** não usa módulo 11. Usa a norma internacional ISO 7064, módulo 97-10, que é outra família
de algoritmo.

**Cartão do SUS** tem rotina própria publicada pelo Ministério da Saúde, com caminhos diferentes para cartão
provisório e definitivo.

**Cartão de pagamento** usa Luhn, que é módulo 10, mais o prefixo do emissor. Luhn sozinho aceita
aproximadamente um em cada dez números compridos, então sem o prefixo você volta ao problema do capítulo 1.

Sete famílias de algoritmo, dezessete tipos de identificador, e cada uma com uma fonte normativa diferente que
precisa ser localizada, lida e conferida.

## A partir daqui, a biblioteca

```{code-cell}
import tarja

print(sorted(tarja.ENTITIES))
```

Dezessete tipos, cada um com a fonte normativa citada no código e a data em que ela foi conferida.

```{code-cell}
print(tarja.validate("BR_CPF",  "529.982.247-25"))
print(tarja.validate("BR_CNPJ", "11.222.333/0001-81"))
print(tarja.validate("BR_CNPJ", "12.ABC.345/01DE-35"))   # alfanumérico
print(tarja.validate("BR_CNJ",  "0000001-83.2017.8.26.0100"))
print(tarja.validate("BR_CNS",  "898 0000 0004 3208"))
```

E achar num texto em vez de validar um valor isolado:

```{code-cell}
texto = (
    "Contrato firmado com a empresa CNPJ 11.222.333/0001-81, "
    "representada pelo sócio de CPF 529.982.247-25, "
    "conforme processo 0000001-83.2017.8.26.0100."
)

for m in tarja.find(texto):
    print(f"{m.entity:10} {m.start:3}:{m.end:<3} score {m.score}")
```

Repare que a saída não mostra os valores. É de propósito, e o capítulo 6 explica por quê.

## O que você ganhou escrevendo à mão

Você não vai reimplementar dezessete algoritmos, e ninguém deveria. O que você ganhou foi outra coisa: agora
consegue ler a implementação de qualquer biblioteca e julgar se ela está certa. Procure a rejeição de dígitos
repetidos. Procure o tratamento do resto igual a dez. Procure se o CNPJ aceita letra. Se as três estiverem lá,
é provável que quem escreveu conhecia o problema. Se faltar alguma, você já sabe o que vai quebrar.

Essa é a única forma honesta de terceirizar uma regra: entendendo-a primeiro.

## O que fica

Dígito verificador é aritmética pública, escrevível em quinze linhas, e é o que separa um CPF de um protocolo.
A regra que não está no algoritmo, a rejeição de dígitos repetidos, é a que mais dá trabalho na prática.

O próximo capítulo trata do caso em que essa aritmética não existe: CEP, placa e telefone não têm dígito
verificador, e aí a única informação disponível é o que está escrito em volta.
