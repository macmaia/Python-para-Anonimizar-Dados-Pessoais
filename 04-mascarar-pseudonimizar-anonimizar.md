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

# 4. Mascarar, pseudonimizar, anonimizar

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

Achado o identificador, o que colocar no lugar dele. Há três respostas, e elas não são graus da mesma coisa.
São decisões diferentes sobre o que você vai conseguir fazer depois.

A pergunta que escolhe entre as três não é "quanto de privacidade eu quero". É: **preciso saber que dois
registros são da mesma pessoa?** e **preciso conseguir voltar ao valor original?**

## Apagar

```{code-cell}
import tarja

doc = "Atendimento de Maria, CPF 529.982.247-25. Reclamacao sobre a fatura."
print(tarja.mask(doc))
```

O valor sumiu. Não dá para reverter, não dá para contar quantas pessoas distintas aparecem, não dá para juntar
com nada. Todo CPF do acervo vira o mesmo rótulo.

É a escolha certa quando o identificador não tem função nenhuma no que vem depois. Você quer ler os chamados
para entender do que as pessoas reclamam, e o número não participa disso.

## Numerar dentro do documento

```{code-cell}
print(tarja.mask("Chamado 3. Titulares 123.456.789-09 e 529.982.247-25.", strategy="pseudonym"))
```

Agora dá para ver que são duas pessoas diferentes, e qual valor apareceu primeiro. O texto continua legível e a
contagem continua possível.

É para isso que a estratégia existe: **uma chamada, vários identificadores**. Dentro daquele texto, rótulos
iguais são o mesmo valor e rótulos diferentes são valores diferentes. Ela cumpre o que promete.

O problema aparece quando alguém usa o resultado fora desse escopo, e isso acontece com facilidade, porque o
jeito natural de processar um acervo é um documento por chamada:

```{code-cell}
a = "Chamado 1. Titular CPF 529.982.247-25."
b = "Chamado 2. Titular CPF 123.456.789-09."   # outra pessoa

print(tarja.mask(a, strategy="pseudonym"))
print(tarja.mask(b, strategy="pseudonym"))
```

**Duas pessoas diferentes, o mesmo rótulo.** A numeração recomeça do 1 a cada chamada, e `_1` significa apenas
"o primeiro CPF deste documento".

Não é defeito, e a documentação não promete o contrário: o rótulo vale **dentro** de uma chamada. O risco é
que o resultado não carrega esse aviso consigo. Um arquivo com dez mil chamados mascarados assim parece
comparável e não é, e um analista que junte por rótulo vai concluir que são a mesma pessoa.

Se o seu pipeline processa documento por documento e alguém vai agregar os resultados depois, esta é a
estratégia que produz o erro mais silencioso dos três.

## Derivar de uma chave

```{code-cell}
import secrets

# A chave é um segredo de verdade, não um texto escolhido a dedo. Em produção ela vem de um gestor de
# segredos, nunca do código. Aqui ela é gerada na hora, então os rótulos abaixo mudam a cada construção
# do livro, e é exatamente isso que se espera de uma chave.
CHAVE = secrets.token_hex(32)

print(tarja.mask(a, strategy="pseudonym_stable", salt=CHAVE))
print(tarja.mask(b, strategy="pseudonym_stable", salt=CHAVE))
```

Rótulos diferentes para pessoas diferentes. E o mesmo valor, em qualquer documento:

```{code-cell}
doc1 = "Atendimento de Maria, CPF 529.982.247-25."
doc2 = "Segundo atendimento, mesmo titular, CPF 529.982.247-25."

print(tarja.mask(doc1, strategy="pseudonym_stable", salt=CHAVE))
print(tarja.mask(doc2, strategy="pseudonym_stable", salt=CHAVE))
```

Mesmo rótulo. É isso que permite montar um conjunto de dados sobre uma pessoa sem o número dela: contar
quantos chamados ela abriu, seguir o caso dela no tempo, cruzar com outro sistema mascarado com a mesma chave.

O rótulo não é aleatório, é calculado a partir do valor e de um segredo. Troque o segredo e tudo muda:

```{code-cell}
print(tarja.mask(doc1, strategy="pseudonym_stable", salt=CHAVE))
print(tarja.mask(doc1, strategy="pseudonym_stable", salt=secrets.token_hex(32)))
```

O mesmo CPF, dois rótulos. Então **a estabilidade vale sob uma chave, e só uma**. Documento mascarado no mês
passado deixa de casar com um mascarado hoje se a chave mudou no meio, e nada quebra, nada avisa: os dados
simplesmente param de se juntar.

O pedaço de quatro caracteres no meio do rótulo é o marcador de geração da chave. Mesma chave, mesmo
marcador. É o que permite perceber que dois rótulos vieram de chaves diferentes em vez de descobrir isso por
uma estatística que não fecha.

## O que isso é, juridicamente

Rótulo estável é **pseudonimização, não anonimização**, e a diferença tem consequência.

Na LGPD, dado anonimizado sai do alcance da lei, porque não se refere mais a pessoa identificável. Dado
pseudonimizado continua dentro, porque a identificação ainda é possível: quem tem a chave reverte, e mesmo sem
a chave o rótulo estável permite seguir uma pessoa por todo o acervo, que é justamente o risco de
reidentificação por cruzamento.

Isso significa que trocar CPF por rótulo estável reduz risco, e não encerra suas obrigações. Base legal,
retenção, segurança e resposta ao titular continuam valendo sobre o dado pseudonimizado.

Se o que você precisa é tirar o dado do alcance da lei, `pseudonym_stable` não faz isso, e nenhuma
substituição reversível faz. Só `redact` chega perto, e mesmo assim o restante do texto pode reidentificar.

## A chave é o dado

Quem tem a chave transforma qualquer rótulo de volta no valor, para todo o universo de CPFs, porque o espaço é
pequeno o bastante para ser enumerado. A chave não é uma configuração, é o próprio dado pessoal noutra forma.

Consequências práticas: a chave não vai no código nem no repositório, vem de um gestor de segredos. Quem tem
acesso à chave tem acesso ao dado, e isso entra no seu controle de acesso. E se a chave vazar, o conjunto
mascarado inteiro vazou junto, retroativamente.

## Escolhendo

| Preciso... | Estratégia |
|---|---|
| ler o texto, o número não importa | `redact` |
| contar pessoas distintas dentro de um documento | `pseudonym` |
| juntar registros da mesma pessoa entre documentos | `pseudonym_stable` |
| poder voltar ao valor original de forma controlada | `Vault`, capítulo 5 |

E uma regra que economiza discussão: escolha a **menos** poderosa que resolve o seu caso. Cada degrau a mais
carrega uma obrigação a mais.

## O que fica

As três não são níveis de proteção, são capacidades diferentes. `pseudonym` não junta e pode enganar quem
supuser que junta. `pseudonym_stable` junta, e por isso continua sendo dado pessoal, sob uma chave que é tão
sensível quanto o dado.

O próximo capítulo trata do caso em que você precisa do valor de volta: mandar o texto para um modelo de
linguagem e reidratar a resposta.
