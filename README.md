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
   <div id="out"><pre><code class="language-html">@model DapperMvcApp.Models.User
 
@{
    ViewData["Title"] = "Новый пользователь";
}
<span class="hljs-tag">&lt;<span class="hljs-title">h2</span>&gt;</span>@ViewData["Title"]<span class="hljs-tag">&lt;/<span class="hljs-title">h2</span>&gt;</span>
 
<span class="hljs-tag">&lt;<span class="hljs-title">form</span> <span class="hljs-attribute">asp-action</span>=<span class="hljs-value">"Create"</span> <span class="hljs-attribute">asp-controller</span>=<span class="hljs-value">"Home"</span>&gt;</span>
    <span class="hljs-tag">&lt;<span class="hljs-title">div</span>&gt;</span>
        <span class="hljs-tag">&lt;<span class="hljs-title">div</span> <span class="hljs-attribute">class</span>=<span class="hljs-value">"form-group"</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">label</span> <span class="hljs-attribute">asp-for</span>=<span class="hljs-value">"Name"</span>&gt;</span><span class="hljs-tag">&lt;/<span class="hljs-title">label</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">input</span> <span class="hljs-attribute">type</span>=<span class="hljs-value">"text"</span> <span class="hljs-attribute">asp-for</span>=<span class="hljs-value">"Name"</span> <span class="hljs-attribute">class</span>=<span class="hljs-value">"form-control form-control-sm"</span> /&gt;</span>
        <span class="hljs-tag">&lt;/<span class="hljs-title">div</span>&gt;</span>
        <span class="hljs-tag">&lt;<span class="hljs-title">div</span> <span class="hljs-attribute">class</span>=<span class="hljs-value">"form-group"</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">label</span> <span class="hljs-attribute">asp-for</span>=<span class="hljs-value">"Age"</span>&gt;</span><span class="hljs-tag">&lt;/<span class="hljs-title">label</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">input</span> <span class="hljs-attribute">asp-for</span>=<span class="hljs-value">"Age"</span> <span class="hljs-attribute">class</span>=<span class="hljs-value">"form-control  form-control-sm"</span> /&gt;</span>
        <span class="hljs-tag">&lt;/<span class="hljs-title">div</span>&gt;</span>
        <span class="hljs-tag">&lt;<span class="hljs-title">div</span> <span class="hljs-attribute">class</span>=<span class="hljs-value">"form-group"</span>&gt;</span>
            <span class="hljs-tag">&lt;<span class="hljs-title">input</span> <span class="hljs-attribute">type</span>=<span class="hljs-value">"submit"</span> <span class="hljs-attribute">value</span>=<span class="hljs-value">"Сохранить"</span> <span class="hljs-attribute">class</span>=<span class="hljs-value">"btn btn-outline-secondary"</span> /&gt;</span>
        <span class="hljs-tag">&lt;/<span class="hljs-title">div</span>&gt;</span>
    <span class="hljs-tag">&lt;/<span class="hljs-title">div</span>&gt;</span>
<span class="hljs-tag">&lt;/<span class="hljs-title">form</span>&gt;</span>
 
<span class="hljs-tag">&lt;<span class="hljs-title">div</span>&gt;</span>
    <span class="hljs-tag">&lt;<span class="hljs-title">a</span> <span class="hljs-attribute">asp-controller</span>=<span class="hljs-value">"Home"</span> <span class="hljs-attribute">asp-action</span>=<span class="hljs-value">"Index"</span>&gt;</span>Вернуться к списку<span class="hljs-tag">&lt;/<span class="hljs-title">a</span>&gt;</span>
<span class="hljs-tag">&lt;/<span class="hljs-title">div</span>&gt;</span>
</code></pre>
</div>
