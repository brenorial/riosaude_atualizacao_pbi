
## 📄 `explicacao_power_query_refresh.md`

# 🔍 Power Query: Conexão com PostgreSQL e Adição de Data de Atualização

Este documento explica passo a passo o código M utilizado no Power Query do Power BI para:

- Conectar-se a um banco PostgreSQL via ODBC;
- Navegar até um `schema` e uma `view` específicos;
- Transformar tipos de colunas;
- Adicionar uma **coluna com a data/hora do último refresh do Power BI** (`DateTime.LocalNow()`).

---

## 🧠 Objetivo do Código

Permitir que os dados carregados a partir do PostgreSQL tragam consigo a informação de **quando foram atualizados no Power BI**, facilitando:

- Auditoria de dashboards;
- Exibição em visuais como "Dados atualizados em...";
- Garantia de que os dados estão atualizados sem depender do agendamento externo.

---

## 🔗 Código Power Query Explicado

```m
let
    // 1️⃣ Etapa: Conectar ao servidor PostgreSQL via ODBC
    Fonte = Odbc.DataSource(
        "Driver={PostgreSQL Unicode(x64)}; Server=10.80.8.32; Port=5432; Database=postgres",
        [HierarchicalNavigation=true]
    ),

    // 2️⃣ Etapa: Acessar o banco de dados 'postgres'
    postgres_Database = Fonte{[Name="postgres", Kind="Database"]}[Data],

    // 3️⃣ Etapa: Acessar o schema 'riosaude'
    riosaude_Schema = postgres_Database{[Name="riosaude", Kind="Schema"]}[Data],

    // 4️⃣ Etapa: Acessar a view 'irs_producao_ambulatorial'
    irs_producao_ambulatorial_View = riosaude_Schema{[Name="irs_producao_ambulatorial", Kind="View"]}[Data],

    // 5️⃣ Etapa: Alterar o tipo da coluna 'fat_paciente_rede_id' para texto
    #"Tipo Alterado" = Table.TransformColumnTypes(
        irs_producao_ambulatorial_View,
        {{"fat_paciente_rede_id", type text}}
    ),

    // 6️⃣ Etapa: Adicionar uma nova coluna com a data/hora atual do refresh
    AdicionarDataAtualizacao = Table.AddColumn(
        #"Tipo Alterado",
        "DataAtualizacao",
        each DateTime.LocalNow(),
        type datetime
    )
in
    AdicionarDataAtualizacao


---

## 📦 Explicações Detalhadas das Etapas

### 1. Conexão via ODBC

```m
Odbc.DataSource("Driver={PostgreSQL Unicode(x64)}; Server=...; Port=...; Database=...", ...)
```

Conecta-se diretamente a um banco PostgreSQL usando o driver ODBC especificado. Os parâmetros são:

* `Server`: endereço IP do servidor PostgreSQL (neste caso, `10.80.8.32`);
* `Port`: porta padrão do PostgreSQL (`5432`);
* `Database`: nome do banco de dados (`postgres`);
* `HierarchicalNavigation=true`: permite a navegação por **schemas**, **tabelas** e **views** como uma estrutura em árvore.

---

### 2. Acesso ao Schema e à View

O Power Query acessa hierarquicamente:

* O banco (`postgres_Database`);
* O schema (`riosaude_Schema`);
* A view final (`irs_producao_ambulatorial_View`).

---

### 3. Transformação de Tipos

```m
Table.TransformColumnTypes(..., {{"fat_paciente_rede_id", type text}})
```

Transforma o tipo da coluna `fat_paciente_rede_id` para texto (`string`) — essencial para garantir consistência no modelo Power BI, relacionamentos e filtragens.

---

### 4. Adição da Coluna `DataAtualizacao`

```m
Table.AddColumn(..., "DataAtualizacao", each DateTime.LocalNow(), type datetime)
```

Essa etapa **insere uma nova coluna** chamada `DataAtualizacao`, preenchida com o valor atual de data e hora no momento em que o Power BI **recarrega os dados**.

* O valor é o mesmo para todas as linhas (não aumenta o tamanho do modelo);
* Pode ser usado para criar **medidas** como:

```dax
Última Atualização = MAX('NomeDaTabela'[DataAtualizacao])
```

---

## 🔁 Como Adaptar para Outro Banco / Schema / View

Se quiser reutilizar esse mesmo modelo de conexão para outras origens, basta **alterar os 3 blocos abaixo**:

| Item              | Linha Atual                                                                                                | O que trocar                                                   |
| ----------------- | ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| 🔸 Banco de dados | `postgres_Database = Fonte{[Name="postgres", Kind="Database"]}[Data],`                                     | Substituir `"postgres"` por `"novo_banco"`                     |
| 🔸 Schema         | `riosaude_Schema = postgres_Database{[Name="riosaude", Kind="Schema"]}[Data],`                             | Substituir `"riosaude"` por `"novo_schema"`                    |
| 🔸 View           | `irs_producao_ambulatorial_View = riosaude_Schema{[Name="irs_producao_ambulatorial", Kind="View"]}[Data],` | Substituir `"irs_producao_ambulatorial"` por `"sua_nova_view"` |

---

## 💡 Dica Extra: Visual no Power BI com Data do Refresh

Crie um **visual de cartão (Card)** com esta medida DAX:

```dax
Última Atualização (Formatada) =
FORMAT(MAX('irs_producao_ambulatorial'[DataAtualizacao]), "dd/MM/yyyy HH:mm:ss")
```

Isso exibirá a informação no seu painel, como:

📆 **Última atualização: 10/09/2025 14:45:13**

---

## ✅ Vantagens dessa Abordagem

* ✔️ Não exige edição no banco de dados (100% Power BI);
* ✔️ A data é atualizada automaticamente no refresh;
* ✔️ Pode ser combinada com filtros, visuais e medidas;
* ✔️ Transparência sobre a **validade dos dados** exibidos.

## Exemplo Completo
```dax
let Fonte = Odbc.DataSource("Driver={PostgreSQL Unicode(x64)}; Server=10.80.8.32; Port=5432; Database=postgres", [HierarchicalNavigation=true]), postgres_Database = Fonte{[Name="postgres",Kind="Database"]}[Data], riosaude_Schema = postgres_Database{[Name="riosaude",Kind="Schema"]}[Data], irs_producao_ambulatorial_View = riosaude_Schema{[Name="irs_producao_ambulatorial",Kind="View"]}[Data], #"Tipo Alterado" = Table.TransformColumnTypes(irs_producao_ambulatorial_View,{{"fat_paciente_rede_id", type text}}) in #"Tipo Alterado"
```
