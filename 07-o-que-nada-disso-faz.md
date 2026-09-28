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

# 7. O que nada disso faz

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

Este capítulo existe porque um livro que só mostra o que funciona não serve para trabalhar. O que vem aqui não
são ressalvas de rodapé, é o contorno da ferramenta, e conhecer o contorno é o que separa usar de confiar.

## Não pega nome, endereço nem dado clínico

```{code-cell}
import tarja

texto = ("Maria Alice Souza, residente na Rua das Laranjeiras 412, apartamento 301, "
         "diagnosticada com hipertensão em consulta de 12/03/2024.")
print(tarja.find(texto))
```

Vazio. Nome, endereço, data e condição de saúde estão todos ali, e nenhum é detectado.

Isso não é lacuna a preencher depois, é outro problema. Identificador estruturado tem regra de formação, e é
por isso que ele pode ser reconhecido com aritmética. Nome não tem: "Souza" é sobrenome e é também o nome de
uma rua e de um município. Endereço não tem. Reconhecer esses depende de modelo de linguagem treinado para a
tarefa, com uma taxa de erro própria, que precisa ser medida separadamente e que é bem pior do que a de
dígito verificador.

Quem precisa dos dois combina duas ferramentas, e mede as duas.

## Ausência de achado não é prova de ausência

Esta é a frase mais importante do livro, e ela vale mesmo com a ferramenta funcionando perfeitamente.

O relatório diz "nenhum identificador encontrado". Isso significa que o detector não achou, e não significa
que não há. Pode haver identificador de um tipo que ele não conhece, escrito num formato que o padrão não
cobre, quebrado por uma quebra de linha no meio, ou num PDF que virou imagem.

Se alguém for usar o seu relatório para afirmar que um acervo está limpo, essa é a frase que precisa estar
escrita no relatório, em português, antes de alguém concluir o contrário.

## Dígito válido não quer dizer que a pessoa existe

```{code-cell}
print(tarja.validate("BR_CPF", "529.982.247-25"))
```

Isso diz que o número é bem formado. Não diz que ele foi emitido, nem a quem. Um valor gerado ao acaso que
feche a aritmética passa exatamente igual, e é assim que se produzem os exemplos deste livro.

O caminho inverso também vale: nada aqui consulta base externa, nada resolve identificador em pessoa. Essa
fronteira é deliberada. Uma ferramenta que confirmasse "este CPF pertence a fulano" seria mais útil para
quem quer proteger e seria imediatamente mais útil ainda para quem quer garimpar.

## Ruído de digitalização derruba a detecção

O teste mais desconfortável é o de caracteres trocados por reconhecimento óptico, o O maiúsculo no lugar do
zero e o l minúsculo no lugar do um.

Na medição publicada do tarja, o subconjunto com esse tipo de ruído fica em **F1 de 0,266**, contra números
acima de 0,95 no texto limpo. Não é degradação, é falha.

O motivo é que o normalizador trata parecidos de Unicode, acento e largura, e **não** trata a confusão entre
letra e dígito, porque fazer isso sem contexto criaria falso positivo em toda parte: nem todo O no meio de
números é um zero.

Consequência prática: se o seu acervo tem documento digitalizado, extraia o texto com uma ferramenta boa e
trate a saída dela como suspeita. Não tome a taxa de acerto em texto limpo como se valesse ali.

## A medição do fornecedor não é a sua

Toda ferramenta publica uma acurácia. Ela foi obtida num conjunto de teste que não é o seu texto.

O capítulo 3 mostrou isso de forma concreta: o limiar de score separa achado bom de ruim **quando o falso
positivo está longe de qualquer palavra de contexto**, o que é propriedade do texto e não do método. Num
acervo de formulário funciona bem, num de prosa corrida funciona pior, e a documentação não tem como avisar
qual é o seu caso.

É por isso que o capítulo 6 existe. Medir no seu texto não é excesso de zelo, é a única forma de saber.

## Nada disso resolve a sua obrigação legal

Achar e mascarar dado pessoal é medida técnica. Continua sendo sua responsabilidade ter base legal para o
tratamento, manter registro das operações, definir e cumprir prazo de retenção, responder ao titular que pede
acesso ou exclusão, e comunicar incidente quando for o caso.

E como o capítulo 4 mostrou, substituir por rótulo estável é pseudonimização: o dado continua pessoal, e
continua dentro da lei.

Uma ferramenta reduz risco. Ela não transfere responsabilidade, e nenhum relatório dela serve como atestado
de conformidade.

## Uso dual

Um detector de identificadores brasileiros é a mesma ferramenta para duas finalidades opostas. Quem quer
proteger usa para achar e mascarar. Quem quer garimpar usa para achar e extrair.

Não existe versão técnica que sirva para uma e não para a outra: a capacidade é a mesma, muda o que se faz com
a saída. É por isso que a biblioteca não resolve identificador em pessoa, não consulta base externa e não
facilita exportar valores em massa, e é por isso que o padrão é esconder o valor em vez de mostrar.

Nada disso impede o mau uso, e não pretende. Desloca o padrão, e deixa o uso indevido como escolha explícita
de quem faz, em vez de acidente de quem não percebeu.

## O que fica

Não pega nome nem endereço. Não prova ausência. Não confirma que alguém existe. Falha em texto digitalizado
com ruído. Não transfere acurácia do fornecedor para o seu caso, nem responsabilidade legal para a ferramenta.

Se depois destes sete capítulos você tem uma noção clara do que consegue afirmar e do que não consegue, o
livro cumpriu o que queria. O resto é medir.

---

Código e documentação do instrumento: [tarja](https://macmaia.github.io/tarja/).
Versão em inglês, para times internacionais:
[Python Cookbook for Brazilian PII](https://macmaia.github.io/Python-Cookbook-for-Brazilian-PII/).
