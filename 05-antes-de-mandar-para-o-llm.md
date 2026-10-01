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
    description: "Como remover dado pessoal do texto antes de enviar para um modelo de linguagem, e como devolver o valor original na resposta sem guardar o valor."
---

# 5. Antes de mandar texto para um modelo de linguagem

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

Este é o caso que mais cresce e o menos tratado. Alguém liga um assistente ao sistema de chamados, ou monta uma
busca sobre a base de documentos, e a partir daquele dia o texto dos clientes sai da sua infraestrutura a cada
requisição, com o que houver dentro dele.

O capítulo anterior terminou nas três estratégias. Nenhuma serve aqui, porque todas elas são de mão única e
neste caso você precisa do valor de volta: o modelo responde falando do titular, e a resposta tem que fazer
sentido para quem a lê.

## Como mascarar antes do LLM e devolver depois?

```{code-cell}
import tarja

texto = "Paciente CPF 529.982.247-25, cartao SUS 898 0000 0004 3208."

cofre = tarja.Vault()
seguro = cofre.protect(texto)
print(seguro)
```

Cada identificador virou um token. O resultado é uma string comum: vai para a API do modelo, para um arquivo ou
para um índice de busca sem tratamento especial.

O modelo trabalha sobre esse texto e responde usando os mesmos tokens:

```{code-cell}
import re

token_cpf = re.search(r"<BR_CPF:[^>]+>", seguro).group()
resposta_do_modelo = f"O titular {token_cpf} deve retornar em 30 dias."
print(resposta_do_modelo)
```

E você reidrata:

```{code-cell}
print(cofre.reveal(resposta_do_modelo, issued_by=seguro))
```

Quatro linhas. O dado pessoal não saiu, e a resposta chegou ao usuário completa.

## Conferir em vez de confiar

Mascarar é fácil de acreditar e difícil de garantir. O identificador que o detector não conhece continua no
texto, e o texto já foi enviado.

```{code-cell}
print("sobrou algo em 'seguro'?", tarja.residual(seguro, report_invalid=True))
print("e aqui?", [m.entity for m in tarja.residual("sobrou um CPF 529.982.247-25 aqui", report_invalid=True)])
```

O `residual()` é uma segunda passada sobre o texto já tratado. Ele ignora os tokens e procura identificador de
verdade. A regra prática: rode antes de enviar, e se voltar algo, não envie.

**O `report_invalid=True` não é opcional aqui.** Sem ele, o `residual()` só relata valor cujo dígito
verificador fecha, então uma sequência de onze dígitos com a cara exata de um CPF e o DV errado volta como
lista vazia, e a lista vazia é lida como permissão para enviar. DV errado quer dizer que o valor não é um CPF
válido. Não quer dizer que não é dado pessoal: é, com a mesma frequência, um erro de digitação num CPF real.
No último portão antes de o texto sair, peça tudo o que tem forma de identificador e decida você. É o mesmo
defeito que o tarja 0.9.0 fechou na própria linha de comando, e vale dizer duas vezes.

Isso não prova que o texto está limpo, prova que o detector não acha mais nada. São coisas diferentes, e o
capítulo 7 insiste nisso.

## Por que o token é preso à chamada

```{code-cell}
outro_cofre = tarja.Vault()
try:
    outro_cofre.reveal(resposta_do_modelo, issued_by=seguro)
except Exception as e:
    print(type(e).__name__)
```

Um cofre não resolve token que ele não emitiu. Parece óbvio, e o caso que importa é mais sutil: num serviço
multiusuário, um cofre só pode atender várias pessoas. Se o `reveal()` resolvesse qualquer token que aquele
cofre já emitiu, bastaria a pessoa A colar num campo de texto um token que apareceu no documento da pessoa B
para receber de volta o dado da pessoa B.

Por isso o escopo é preso à chamada de `protect()` que emitiu os tokens, e por isso o escopo é de uso único:

```{code-cell}
try:
    cofre.reveal(resposta_do_modelo, issued_by=seguro)
except Exception as e:
    print(type(e).__name__)
```

A segunda tentativa no mesmo escopo é recusada. Use `reuse=True` quando a repetição for legítima, sabendo que
você está abrindo mão de um controle.

## O modelo estraga o token

Este é o problema que aparece em produção e não em laboratório.

Um modelo de linguagem não trata o token como um símbolo intocável. Ele quebra a linha no meio, muda a caixa
dos caracteres, às vezes parafraseia. O `reveal()` compara o token, e token alterado não é encontrado:

```{code-cell}
quebrado = token_cpf[:20] + "\n" + token_cpf[20:]
print(repr(quebrado))
print("resolve?", "529.982.247-25" in cofre.reveal(f"Titular {quebrado}.", issued_by=seguro, reuse=True))
```

Resolve. Espaço em branco e caixa não carregam informação, então aceitá-los custa **zero** em segurança: os
caracteres hexadecimais continuam tendo que bater exatamente, e quem quisesse forjar um token não ficou um
passo mais perto.

E é aí que a tolerância para. Token com um dígito faltando ou trocado não resolve:

```{code-cell}
faltando = token_cpf[:-3] + ">"
print("resolve?", "529.982.247-25" in cofre.reveal(f"Titular {faltando}.", issued_by=seguro, reuse=True))
```

Cada caractere de tolerância a mais é um caractere a menos de segredo, e é assim que se constrói um oráculo
por acidente: alguém que possa tentar muitas vezes acaba acertando um token que resolve.

Duas coisas a fazer mesmo assim. Instrua o modelo, no prompt, a repetir os tokens exatamente como recebidos, o
que reduz o problema na origem. E trate como erro do seu lado a resposta que ainda contém token depois da
reidratação, sem mostrar ao usuário.

## O que a biblioteca não faz

O mapa de token para valor vive na memória de um processo. Isso tem três consequências que você precisa saber
antes de colocar em produção, e não depois.

Ele morre com o processo, então um trabalho de ingestão não consegue passar tokens para um serviço que roda
separado. Ele não atravessa réplicas: com duas instâncias atrás de um balanceador, o `protect()` e o `reveal()`
do mesmo documento têm que cair na mesma. E não existe registro de quem reverteu o quê.

Nada disso é descuido. Guardar esse mapa de forma durável puxa junto guarda de chave, controle de acesso,
retenção, exclusão a pedido, backup e auditoria, que são decisões sobre o risco de quem opera, e não padrão
razoável de biblioteca. Quem tem o objeto `Vault` na mão reverte tudo, do mesmo jeito que quem tem a chave.

## O que fica

Tokenizar, enviar, reidratar e conferir o resíduo resolve o caso de uso mais comum de dado pessoal indo para
fora hoje, em quatro linhas. O escopo preso à chamada é o que impede um usuário de reverter o dado de outro. E
o modelo vai estragar o token de vez em quando, o que é um defeito visível e não um vazamento.

O próximo capítulo é sobre medir: quanto disso está funcionando, e como saber sem acreditar em ninguém.
