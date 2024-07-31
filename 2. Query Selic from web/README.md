# Sobre
Esta planilha consulta os dados da SELIC diretamente do site da [RFB](https://www.gov.br/receitafederal/pt-br/assuntos/orientacao-tributaria/pagamentos-e-parcelamentos/taxa-de-juros-selic).

## Obtenção dos dados

Para a captura, a planilha segue as seguitnes etapas através do Power Query:

1. Obterm o HTML completo utilizando `Web.BrowserContents`
1. Remove as partes do HTML que não contém as informações da SELIC utilizando `Text.Split`.
1. Usando a mesma função, separa as tabelas presentes no site para cada ano e converte a lista em tabela usando `Table.FromList`.
1. Faz uma nova higienização dos dados, removendo um cabeçalho com `Table.Skip`.
1. E para cada linha de código HTML da tabela, adiciona uma coluna chamando uma função personalizada onde o seguinte é realizado:
   1. Obtem os dados da tabela HTML usando a função `Html.Table`.
   1. Promove o cabeçalho `Table.PromoteHeaders`.
   1. Transforma colunas em linhas com `Table.UnpivotOtherColumns`.
1. Expande o resultado desta nova coluna criada
1. A partir do Mês e ano em português, obtem o data
1. Remove colunas não importantes
1. Ordena os dados
1. Altera o tipo de dados do valor da Selic
1. Remove linhas vazias
1. Adiciona uma coluna extra para facilitar o cálculo da selic como juros compostos.

## Código final:

Função personalizada
```
let
    Fonte = (HTMLCode as any) => let
        Colunas = List.Transform(List.Generate(() => 1, each _ < 11, each _ + 1), each {"Column" & Text.From(_), "TABLE:nth-child(1) > * > TR > :nth-child(" & Text.From(_) & ")"}),
        Fonte = Html.Table(HTMLCode, Colunas, [RowSelector="tr"]),
        #"Cabeçalhos Promovidos" = Table.PromoteHeaders(Fonte, [PromoteAllScalars=true]),
        #"Outras Colunas Não Dinâmicas" = Table.UnpivotOtherColumns(#"Cabeçalhos Promovidos", {"Mês/Ano"}, "Ano", "Valor")
    in
        #"Outras Colunas Não Dinâmicas"
in
    Fonte
```
Obtenção de dados da Selic:
```
let
    Fonte = Web.BrowserContents("https://www.gov.br/receitafederal/pt-br/assuntos/orientacao-tributaria/pagamentos-e-parcelamentos/taxa-de-juros-selic"),
    Personalizar1 = Text.Split(Fonte,"<b><a name=""Selicmensalmente"" id=""Selicmensalmente""></a>Taxa de Juros Selic Acumulada Mensalmente</b>"),
    RemoverAcumulado = Personalizar1{0},
    RemoverCabecalho = Text.Split(RemoverAcumulado, "name=""Taxa_de_Juros_Selic"" id=""Taxa_de_Juros_Selic""></a><a name=""Taxaselic"" id=""Taxaselic"">"){1},
    Personalizar2 = Text.Split(Text.Replace(RemoverCabecalho,"<table class=", "@@@@@<table class="), "@@@@@"),
    #"Convertido para Tabela" = Table.FromList(Personalizar2, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Linhas Superiores Removidas" = Table.Skip(#"Convertido para Tabela",1),
    #"Personalização Adicionada" = Table.AddColumn(#"Linhas Superiores Removidas", "Tabela", each GetHTML([Column1])),
    #"Tabela Expandido" = Table.ExpandTableColumn(#"Personalização Adicionada", "Tabela", {"Mês/Ano", "Ano", "Valor"}, {"Mês/Ano", "Ano", "Valor"}),
    #"Personalização Adicionada1" = Table.AddColumn(#"Tabela Expandido", "Data", each Date.From("1 - " & [#"Mês/Ano"] & " - " & [Ano], "pt-BR"), type date),
    #"Outras Colunas Removidas1" = Table.SelectColumns(#"Personalização Adicionada1",{"Data", "Valor"}),
    #"Linhas Classificadas" = Table.Sort(#"Outras Colunas Removidas1",{{"Data", Order.Ascending}}),
    #"Tipo Alterado" = Table.TransformColumnTypes(#"Linhas Classificadas",{{"Valor", Percentage.Type}}),
    #"Linhas Filtradas" = Table.SelectRows(#"Tipo Alterado", each ([Valor] <> null)),
    #"Personalização Adicionada2" = Table.AddColumn(#"Linhas Filtradas", "HideColumn", each 1+[Valor])
in
    #"Personalização Adicionada2"
```


## Cálculo

Para usar a selic recem obtida (considerando que o nome da tabela é Selic), as seguintes fórmulas podem ser utilizadas:

Juros Simples
```
=SOMASES(Selic[Valor];Selic[Data];">="&INICIO;Selic[Data];"<="&FIM)
```
Juros compostos
```
=MULT(FILTRO(Selic[HideColumn];(Selic[Data]>=INICIO)*(Selic[Data]<=FIM)))-1
```

Onde "INICIO" e "FIM" são as datas iniciais e finais respectivamente.