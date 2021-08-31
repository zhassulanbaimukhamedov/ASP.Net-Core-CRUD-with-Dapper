# ASP.Net-Core-CRUD-with-Dapper
1. Создадим новый ASP.NET Core MVC приложение. (Let's create a new ASP.NET Core MVC application)
2. Установим через Nuget пакеты Microsoft.Data.SqlClient и Dapper. (Install Microsoft.Data.SqlClient and Dapper packages via Nuget)
3. Определим модель User, с которым будем работать. (Define the User model, with wich we will work)
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public int Age { get; set; }
    }

5. Создадим базу userstore и таблицу Users. (Create database userstore, and Users table)
  create database userstore;
  create table Users 
  (
    Id int not null auto_increment, 
    Name nvarchar(5) not null, 
    Age int not null,
    primary_key (Id)
  )

