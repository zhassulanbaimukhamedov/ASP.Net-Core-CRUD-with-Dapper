# ASP.Net-Core-CRUD-with-Dapper
1. Создадим новый ASP.NET Core MVC приложение. (Let's create a new ASP.NET Core MVC application)
2. Установим через Nuget пакеты Microsoft.Data.SqlClient и Dapper. (Install Microsoft.Data.SqlClient and Dapper packages via Nuget)
3. Определим модель User, с которым будем работать. (Define the User model, with wich we will work)
<pre><code class="has-line-data" data-line-start="1" data-line-end="9">    public class User
    {
        public int Id { get; <span class="hljs-built_in">set</span>; }
        public string Name { get; <span class="hljs-built_in">set</span>; }
        public int Age { get; <span class="hljs-built_in">set</span>; }
    }

</code></pre>
4. Создадим базу userstore и таблицу Users. (Create database userstore, and Users table)
 <pre><code class="has-line-data" data-line-start="1" data-line-end="11">create database userstore;
create table Users 
  (
    Id int not null auto_increment, 
    Name nvarchar(<span class="hljs-number">5</span>) not null, 
    Age int not null,
    primary_key (Id)
  )

</code></pre>
