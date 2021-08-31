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

5. В папку Models добавим интерфейс и класс репозитория, через который будем работать с базой данных. (Add interface and class of repository to the Models folder, for integrate with database)
    Dapper предоставляет метод расширения Query<T> для объектов IDbConnection, для осуществления запросов , который в качестве параметра принимает sql-выражение и может возвращать объект типа T, с которым сопоставляются результаты запроса.

<pre><code class="has-line-data" data-line-start="1" data-line-end="66">    public interface IUserRepository
    {
        void Create(User user);
        void Delete(int id);
        User Get(int id);
        List&lt;User&gt; GetUsers();
        void Update(User user);
    }
    public class UserRepository : IUserRepository
    {
        string connectionString = null;
        public UserRepository(string conn)
        {
            connectionString = conn;
        }
        public List&lt;User&gt; <span class="hljs-function"><span class="hljs-title">GetUsers</span></span>()
        {
            using (IDbConnection db = new SqlConnection(connectionString))
            {
                <span class="hljs-built_in">return</span> db.Query&lt;User&gt;(<span class="hljs-string">"SELECT * FROM Users"</span>).ToList();
            }
        }
 
        public User Get(int id)
        {
            using (IDbConnection db = new SqlConnection(connectionString))
            {
                <span class="hljs-built_in">return</span> db.Query&lt;User&gt;(<span class="hljs-string">"SELECT * FROM Users WHERE Id = @id"</span>, new { id }).FirstOrDefault();
            }
        }
 
        public void Create(User user)
        {
            using (IDbConnection db = new SqlConnection(connectionString))
            {
                var sqlQuery = <span class="hljs-string">"INSERT INTO Users (Name, Age) VALUES(@Name, @Age)"</span>;
                db.Execute(sqlQuery, user);
 
                // если мы хотим получить id добавленного пользователя
                //var sqlQuery = <span class="hljs-string">"INSERT INTO Users (Name, Age) VALUES(@Name, @Age); SELECT CAST(SCOPE_IDENTITY() as int)"</span>;
                //int? userId = db.Query&lt;int&gt;(sqlQuery, user).FirstOrDefault();
                //user.Id = userId.Value;
            }
        }
 
        public void Update(User user)
        {
            using (IDbConnection db = new SqlConnection(connectionString))
            {
                var sqlQuery = <span class="hljs-string">"UPDATE Users SET Name = @Name, Age = @Age WHERE Id = @Id"</span>;
                db.Execute(sqlQuery, user);
            }
        }
 
        public void Delete(int id)
        {
            using (IDbConnection db = new SqlConnection(connectionString))
            {
                var sqlQuery = <span class="hljs-string">"DELETE FROM Users WHERE Id = @id"</span>;
                db.Execute(sqlQuery, new { id });
            }
        }
    }

</code></pre>

6. Далее изменим Startup (Next, let's change Startup)
    С помощью механизма внедрения зависимостей здесь устанавливается зависимость для интерфейса IUserRepository в виде объекта UserRepository, в конструктор которого передается строка подключения к бд.
    
    <pre><code class="has-line-data" data-line-start="1" data-line-end="8">    public void ConfigureServices(IServiceCollection services)
        {
            string connectionString = <span class="hljs-string">"Server=.\\SQLEXPRESS;Initial Catalog=userstore;Integrated Security=True"</span>;
            services.AddTransient&lt;IUserRepository, UserRepository&gt;(provider =&gt; new UserRepository(connectionString));
            services.AddControllersWithViews();
        }
</code></pre>
    
7. Поменяем HomeController для взаимодействия с бд и выполнения CRUD-операции. (Let's change HomeController for interaction with the database and performing a CRUD-operation)
    <pre><code class="has-line-data" data-line-start="1" data-line-end="65"> public class HomeController : Controller
    {
        IUserRepository repo;
        public HomeController(IUserRepository r)
        {
            repo = r;
        }
        public ActionResult <span class="hljs-function"><span class="hljs-title">Index</span></span>()
        {
            <span class="hljs-built_in">return</span> View(repo.GetUsers());
        }
 
        public ActionResult Details(int id)
        {
            User user = repo.Get(id);
            <span class="hljs-keyword">if</span> (user != null)
                <span class="hljs-built_in">return</span> View(user);
            <span class="hljs-built_in">return</span> NotFound();
        }
 
        public ActionResult <span class="hljs-function"><span class="hljs-title">Create</span></span>()
        {
            <span class="hljs-built_in">return</span> View();
        }
 
        [HttpPost]
        public ActionResult Create(User user)
        {
            repo.Create(user);
            <span class="hljs-built_in">return</span> RedirectToAction(<span class="hljs-string">"Index"</span>);
        }
 
        public ActionResult Edit(int id)
        {
            User user = repo.Get(id);
            <span class="hljs-keyword">if</span> (user != null)
                <span class="hljs-built_in">return</span> View(user);
            <span class="hljs-built_in">return</span> NotFound();
        }
 
        [HttpPost]
        public ActionResult Edit(User user)
        {
            repo.Update(user);
            <span class="hljs-built_in">return</span> RedirectToAction(<span class="hljs-string">"Index"</span>);
        }
 
        [HttpGet]
        [ActionName(<span class="hljs-string">"Delete"</span>)]
        public ActionResult ConfirmDelete(int id)
        {
            User user = repo.Get(id);
            <span class="hljs-keyword">if</span> (user != null)
                <span class="hljs-built_in">return</span> View(user);
            <span class="hljs-built_in">return</span> NotFound();
        }
        [HttpPost]
        public ActionResult Delete(int id)
        {
            repo.Delete(id);
            <span class="hljs-built_in">return</span> RedirectToAction(<span class="hljs-string">"Index"</span>);
        }
    }
</code></pre>

8. Добавим представления следующим образом. (Let's add view as follows)
    Index.cshtml
 <pre><code class="language-html">@model IEnumerable<span class="hljs-tag">&lt;<span class="hljs-title">DapperMvcApp.Models.User</span>&gt;</span>
@{
    ViewData["Title"] = "Список пользователей";
}
<span class="hljs-tag">&lt;<span class="hljs-title">h2</span>&gt;</span>@ViewData["Title"]<span class="hljs-tag">&lt;/<span class="hljs-title">h2</span>&gt;</span>
<span class="hljs-tag">&lt;<span class="hljs-title">p</span>&gt;</span>
    <span class="hljs-tag">&lt;<span class="hljs-title">a</span> <span class="hljs-attribute">asp-controller</span>=<span class="hljs-value">"Home"</span> <span class="hljs-attribute">asp-action</span>=<span class="hljs-value">"Create"</span>&gt;</span>Добавить<span class="hljs-tag">&lt;/<span class="hljs-title">a</span>&gt;</span>
<span class="hljs-tag">&lt;/<span class="hljs-title">p</span>&gt;</span>
<span class="hljs-tag">&lt;<span class="hljs-title">table</span> <span class="hljs-attribute">class</span>=<span class="hljs-value">"table"</span>&gt;</span>
    <span class="hljs-tag">&lt;<span class="hljs-title">tr</span>&gt;</span>
        <span class="hljs-tag">&lt;<span class="hljs-title">th</span>&gt;</span>Имя<span class="hljs-tag">&lt;/<span class="hljs-title">th</span>&gt;</span>
        <span class="hljs-tag">&lt;<span class="hljs-title">th</span>&gt;</span>Возраст<span class="hljs-tag">&lt;/<span class="hljs-title">th</span>&gt;</span>
        <span class="hljs-tag">&lt;<span class="hljs-title">th</span>&gt;</span><span class="hljs-tag">&lt;/<span class="hljs-title">th</span>&gt;</span>
    <span class="hljs-tag">&lt;/<span class="hljs-title">tr</span>&gt;</span>
 
    @foreach (var item in Model)
    {
        <span class="hljs-tag">&lt;<span class="hljs-title">tr</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">td</span>&gt;</span>@item.Name<span class="hljs-tag">&lt;/<span class="hljs-title">td</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">td</span>&gt;</span>
                @item.Age
            <span class="hljs-tag">&lt;/<span class="hljs-title">td</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">td</span>&gt;</span>
                <span class="hljs-tag">&lt;<span class="hljs-title">a</span> <span class="hljs-attribute">asp-controller</span>=<span class="hljs-value">"Home"</span> <span class="hljs-attribute">asp-action</span>=<span class="hljs-value">"Edit"</span> <span class="hljs-attribute">asp-route-id</span>=<span class="hljs-value">"@item.Id"</span>&gt;</span>Изменить<span class="hljs-tag">&lt;/<span class="hljs-title">a</span>&gt;</span> |
                <span class="hljs-tag">&lt;<span class="hljs-title">a</span> <span class="hljs-attribute">asp-controller</span>=<span class="hljs-value">"Home"</span> <span class="hljs-attribute">asp-action</span>=<span class="hljs-value">"Delete"</span> <span class="hljs-attribute">asp-route-id</span>=<span class="hljs-value">"@item.Id"</span>&gt;</span>Удалить<span class="hljs-tag">&lt;/<span class="hljs-title">a</span>&gt;</span>
            <span class="hljs-tag">&lt;/<span class="hljs-title">td</span>&gt;</span>
        <span class="hljs-tag">&lt;/<span class="hljs-title">tr</span>&gt;</span>
    }
</code></pre>
    
9. Create.cshtml
   <div class="container"><div class="line number1 index0 alt2"><code class="xml plain">@model DapperMvcApp.Models.User</code></div><div class="line number2 index1 alt1">&nbsp;</div><div class="line number3 index2 alt2"><code class="xml plain">@{</code></div><div class="line number4 index3 alt1"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">ViewData["Title"] = "Новый пользователь";</code></div><div class="line number5 index4 alt2"><code class="xml plain">}</code></div><div class="line number6 index5 alt1"><code class="xml plain">&lt;</code><code class="xml keyword">h2</code><code class="xml plain">&gt;@ViewData["Title"]&lt;/</code><code class="xml keyword">h2</code><code class="xml plain">&gt;</code></div><div class="line number7 index6 alt2">&nbsp;</div><div class="line number8 index7 alt1"><code class="xml plain">&lt;</code><code class="xml keyword">form</code> <code class="xml color1">asp-action</code><code class="xml plain">=</code><code class="xml string">"Create"</code> <code class="xml color1">asp-controller</code><code class="xml plain">=</code><code class="xml string">"Home"</code><code class="xml plain">&gt;</code></div><div class="line number9 index8 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">div</code><code class="xml plain">&gt;</code></div><div class="line number10 index9 alt1"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">div</code> <code class="xml color1">class</code><code class="xml plain">=</code><code class="xml string">"form-group"</code><code class="xml plain">&gt;</code></div><div class="line number11 index10 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">label</code> <code class="xml color1">asp-for</code><code class="xml plain">=</code><code class="xml string">"Name"</code><code class="xml plain">&gt;&lt;/</code><code class="xml keyword">label</code><code class="xml plain">&gt;</code></div><div class="line number12 index11 alt1"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">input</code> <code class="xml color1">type</code><code class="xml plain">=</code><code class="xml string">"text"</code> <code class="xml color1">asp-for</code><code class="xml plain">=</code><code class="xml string">"Name"</code> <code class="xml color1">class</code><code class="xml plain">=</code><code class="xml string">"form-control form-control-sm"</code> <code class="xml plain">/&gt;</code></div><div class="line number13 index12 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;/</code><code class="xml keyword">div</code><code class="xml plain">&gt;</code></div><div class="line number14 index13 alt1"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">div</code> <code class="xml color1">class</code><code class="xml plain">=</code><code class="xml string">"form-group"</code><code class="xml plain">&gt;</code></div><div class="line number15 index14 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">label</code> <code class="xml color1">asp-for</code><code class="xml plain">=</code><code class="xml string">"Age"</code><code class="xml plain">&gt;&lt;/</code><code class="xml keyword">label</code><code class="xml plain">&gt;</code></div><div class="line number16 index15 alt1"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">input</code> <code class="xml color1">asp-for</code><code class="xml plain">=</code><code class="xml string">"Age"</code> <code class="xml color1">class</code><code class="xml plain">=</code><code class="xml string">"form-control&nbsp; form-control-sm"</code> <code class="xml plain">/&gt;</code></div><div class="line number17 index16 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;/</code><code class="xml keyword">div</code><code class="xml plain">&gt;</code></div><div class="line number18 index17 alt1"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">div</code> <code class="xml color1">class</code><code class="xml plain">=</code><code class="xml string">"form-group"</code><code class="xml plain">&gt;</code></div><div class="line number19 index18 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">input</code> <code class="xml color1">type</code><code class="xml plain">=</code><code class="xml string">"submit"</code> <code class="xml color1">value</code><code class="xml plain">=</code><code class="xml string">"Сохранить"</code> <code class="xml color1">class</code><code class="xml plain">=</code><code class="xml string">"btn btn-outline-secondary"</code> <code class="xml plain">/&gt;</code></div><div class="line number20 index19 alt1"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;/</code><code class="xml keyword">div</code><code class="xml plain">&gt;</code></div><div class="line number21 index20 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;/</code><code class="xml keyword">div</code><code class="xml plain">&gt;</code></div><div class="line number22 index21 alt1"><code class="xml plain">&lt;/</code><code class="xml keyword">form</code><code class="xml plain">&gt;</code></div><div class="line number23 index22 alt2">&nbsp;</div><div class="line number24 index23 alt1"><code class="xml plain">&lt;</code><code class="xml keyword">div</code><code class="xml plain">&gt;</code></div><div class="line number25 index24 alt2"><code class="xml spaces">&nbsp;&nbsp;&nbsp;&nbsp;</code><code class="xml plain">&lt;</code><code class="xml keyword">a</code> <code class="xml color1">asp-controller</code><code class="xml plain">=</code><code class="xml string">"Home"</code> <code class="xml color1">asp-action</code><code class="xml plain">=</code><code class="xml string">"Index"</code><code class="xml plain">&gt;Вернуться к списку&lt;/</code><code class="xml keyword">a</code><code class="xml plain">&gt;</code></div><div class="line number26 index25 alt1"><code class="xml plain">&lt;/</code><code class="xml keyword">div</code><code class="xml plain">&gt;</code></div></div>
    
