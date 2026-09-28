# Python para Anonimizar Dados Pessoais

Livro aberto sobre encontrar e tratar dado pessoal em texto em português, dentro de um sistema que já está
rodando. Escrito para quem programa.

Leia em **https://macmaia.github.io/Python-para-Anonimizar-Dados-Pessoais/**

Versão em inglês, voltada a times internacionais que processam dado de cliente brasileiro:
[Python-Cookbook-for-Brazilian-PII](https://github.com/macmaia/Python-Cookbook-for-Brazilian-PII)

O instrumento usado ao longo do livro é o [tarja](https://github.com/macmaia/tarja), biblioteca aberta sob
Apache 2.0.

## construir localmente

```bash
pip install -r requirements.txt
jupyter-book build .
open _build/html/index.html
```

O código de cada capítulo é executado na construção, então a saída impressa nas páginas é real. Se uma célula
quebrar, a construção falha, o que é o comportamento desejado num livro que ensina a rodar código.

## licença

Texto sob [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.pt-br). Código sob Apache 2.0.

Escrito por [Maria Alice Maia](https://github.com/macmaia) · tarja@micah6ai.com
