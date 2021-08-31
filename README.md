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
  <pre class="prettyprint linenums prettyprinted" style=""><ol class="linenums"><li class="L0"><code class="lang-html"><span class="pln">@model IEnumerable</span><span class="tag">&lt;DapperMvcApp.Models.User&gt;</span></code></li><li class="L1"><code class="lang-html"><span class="pln">@{</span></code></li><li class="L2"><code class="lang-html"><span class="pln">    ViewData["Title"] = "Список пользователей";</span></code></li><li class="L3"><code class="lang-html"><span class="pln">}</span></code></li><li class="L4"><code class="lang-html"><span class="tag">&lt;h2&gt;</span><span class="pln">@ViewData["Title"]</span><span class="tag">&lt;/h2&gt;</span></code></li><li class="L5"><code class="lang-html"><span class="tag">&lt;p&gt;</span></code></li><li class="L6"><code class="lang-html"><span class="pln">    </span><span class="tag">&lt;a</span><span class="pln"> </span><span class="atn">asp-controller</span><span class="pun">=</span><span class="atv">"Home"</span><span class="pln"> </span><span class="atn">asp-action</span><span class="pun">=</span><span class="atv">"Create"</span><span class="tag">&gt;</span><span class="pln">Добавить</span><span class="tag">&lt;/a&gt;</span></code></li><li class="L7"><code class="lang-html"><span class="tag">&lt;/p&gt;</span></code></li><li class="L8"><code class="lang-html"><span class="tag">&lt;table</span><span class="pln"> </span><span class="atn">class</span><span class="pun">=</span><span class="atv">"table"</span><span class="tag">&gt;</span></code></li><li class="L9"><code class="lang-html"><span class="pln">    </span><span class="tag">&lt;tr&gt;</span></code></li><li class="L0"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;th&gt;</span><span class="pln">Имя</span><span class="tag">&lt;/th&gt;</span></code></li><li class="L1"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;th&gt;</span><span class="pln">Возраст</span><span class="tag">&lt;/th&gt;</span></code></li><li class="L2"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;th&gt;&lt;/th&gt;</span></code></li><li class="L3"><code class="lang-html"><span class="pln">    </span><span class="tag">&lt;/tr&gt;</span></code></li><li class="L4"><code class="lang-html"></code></li><li class="L5"><code class="lang-html"><span class="pln">    @foreach (var item in Model)</span></code></li><li class="L6"><code class="lang-html"><span class="pln">    {</span></code></li><li class="L7"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;tr&gt;</span></code></li><li class="L8"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;td&gt;</span><span class="pln">@item.Name</span><span class="tag">&lt;/td&gt;</span></code></li><li class="L9"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;td&gt;</span></code></li><li class="L0"><code class="lang-html"><span class="pln">                @item.Age</span></code></li><li class="L1"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;/td&gt;</span></code></li><li class="L2"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;td&gt;</span></code></li><li class="L3"><code class="lang-html"><span class="pln">                </span><span class="tag">&lt;a</span><span class="pln"> </span><span class="atn">asp-controller</span><span class="pun">=</span><span class="atv">"Home"</span><span class="pln"> </span><span class="atn">asp-action</span><span class="pun">=</span><span class="atv">"Edit"</span><span class="pln"> </span><span class="atn">asp-route-id</span><span class="pun">=</span><span class="atv">"@item.Id"</span><span class="tag">&gt;</span><span class="pln">Изменить</span><span class="tag">&lt;/a&gt;</span><span class="pln"> |</span></code></li><li class="L4"><code class="lang-html"><span class="pln">                </span><span class="tag">&lt;a</span><span class="pln"> </span><span class="atn">asp-controller</span><span class="pun">=</span><span class="atv">"Home"</span><span class="pln"> </span><span class="atn">asp-action</span><span class="pun">=</span><span class="atv">"Delete"</span><span class="pln"> </span><span class="atn">asp-route-id</span><span class="pun">=</span><span class="atv">"@item.Id"</span><span class="tag">&gt;</span><span class="pln">Удалить</span><span class="tag">&lt;/a&gt;</span></code></li><li class="L5"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;/td&gt;</span></code></li><li class="L6"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;/tr&gt;</span></code></li><li class="L7"><code class="lang-html"><span class="pln">    }</span></code></li><li class="L8"><code class="lang-html"><span class="tag">&lt;/table&gt;</span></code></li></ol></pre>

    
    
    
    
    
    
    
    
9. Create.cshtml
    
    <pre class="prettyprint linenums prettyprinted" style=""><ol class="linenums"><li class="L0"><code class="lang-html"><span class="pln">@model DapperMvcApp.Models.User</span></code></li><li class="L1"><code class="lang-html"></code></li><li class="L2"><code class="lang-html"><span class="pln">@{</span></code></li><li class="L3"><code class="lang-html"><span class="pln">    ViewData["Title"] = "Новый пользователь";</span></code></li><li class="L4"><code class="lang-html"><span class="pln">}</span></code></li><li class="L5"><code class="lang-html"><span class="tag">&lt;h2&gt;</span><span class="pln">@ViewData["Title"]</span><span class="tag">&lt;/h2&gt;</span></code></li><li class="L6"><code class="lang-html"></code></li><li class="L7"><code class="lang-html"><span class="tag">&lt;form</span><span class="pln"> </span><span class="atn">asp-action</span><span class="pun">=</span><span class="atv">"Create"</span><span class="pln"> </span><span class="atn">asp-controller</span><span class="pun">=</span><span class="atv">"Home"</span><span class="tag">&gt;</span></code></li><li class="L8"><code class="lang-html"><span class="pln">    </span><span class="tag">&lt;div&gt;</span></code></li><li class="L9"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;div</span><span class="pln"> </span><span class="atn">class</span><span class="pun">=</span><span class="atv">"form-group"</span><span class="tag">&gt;</span></code></li><li class="L0"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;label</span><span class="pln"> </span><span class="atn">asp-for</span><span class="pun">=</span><span class="atv">"Name"</span><span class="tag">&gt;&lt;/label&gt;</span></code></li><li class="L1"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;input</span><span class="pln"> </span><span class="atn">type</span><span class="pun">=</span><span class="atv">"text"</span><span class="pln"> </span><span class="atn">asp-for</span><span class="pun">=</span><span class="atv">"Name"</span><span class="pln"> </span><span class="atn">class</span><span class="pun">=</span><span class="atv">"form-control form-control-sm"</span><span class="pln"> </span><span class="tag">/&gt;</span></code></li><li class="L2"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;/div&gt;</span></code></li><li class="L3"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;div</span><span class="pln"> </span><span class="atn">class</span><span class="pun">=</span><span class="atv">"form-group"</span><span class="tag">&gt;</span></code></li><li class="L4"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;label</span><span class="pln"> </span><span class="atn">asp-for</span><span class="pun">=</span><span class="atv">"Age"</span><span class="tag">&gt;&lt;/label&gt;</span></code></li><li class="L5"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;input</span><span class="pln"> </span><span class="atn">asp-for</span><span class="pun">=</span><span class="atv">"Age"</span><span class="pln"> </span><span class="atn">class</span><span class="pun">=</span><span class="atv">"form-control  form-control-sm"</span><span class="pln"> </span><span class="tag">/&gt;</span></code></li><li class="L6"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;/div&gt;</span></code></li><li class="L7"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;div</span><span class="pln"> </span><span class="atn">class</span><span class="pun">=</span><span class="atv">"form-group"</span><span class="tag">&gt;</span></code></li><li class="L8"><code class="lang-html"><span class="pln">            </span><span class="tag">&lt;input</span><span class="pln"> </span><span class="atn">type</span><span class="pun">=</span><span class="atv">"submit"</span><span class="pln"> </span><span class="atn">value</span><span class="pun">=</span><span class="atv">"Сохранить"</span><span class="pln"> </span><span class="atn">class</span><span class="pun">=</span><span class="atv">"btn btn-outline-secondary"</span><span class="pln"> </span><span class="tag">/&gt;</span></code></li><li class="L9"><code class="lang-html"><span class="pln">        </span><span class="tag">&lt;/div&gt;</span></code></li><li class="L0"><code class="lang-html"><span class="pln">    </span><span class="tag">&lt;/div&gt;</span></code></li><li class="L1"><code class="lang-html"><span class="tag">&lt;/form&gt;</span></code></li><li class="L2"><code class="lang-html"></code></li><li class="L3"><code class="lang-html"><span class="tag">&lt;div&gt;</span></code></li><li class="L4"><code class="lang-html"><span class="pln">    </span><span class="tag">&lt;a</span><span class="pln"> </span><span class="atn">asp-controller</span><span class="pun">=</span><span class="atv">"Home"</span><span class="pln"> </span><span class="atn">asp-action</span><span class="pun">=</span><span class="atv">"Index"</span><span class="tag">&gt;</span><span class="pln">Вернуться к списку</span><span class="tag">&lt;/a&gt;</span></code></li><li class="L5"><code class="lang-html"><span class="tag">&lt;/div&gt;</span></code></li></ol></pre>
    
    
    
