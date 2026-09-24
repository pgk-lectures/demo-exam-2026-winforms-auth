# Авторизация в Windows Forms с Entity Framework Core

Этот вариант показывает, как сделать авторизацию для приложения из **demo-exam-2026** с использованием **Entity Framework Core (EF Core)**.

В результате приложение будет:

- проверять логин и пароль по данным базы;
- определять роль пользователя;
- открывать основной интерфейс после успешного входа;
- показывать ФИО пользователя;
- позволять войти как гость;
- возвращать пользователя обратно на окно авторизации.

> Важно: в задании ДЭ база данных создаётся в модуле 1. Здесь мы **не создаём новую базу**, а подключаем уже существующую.

## 1. Что должно быть в базе

В исходных материалах ДЭ есть файл **user_import.xlsx** с пользователями. В итоговой базе у вас должна быть таблица пользователей.

Для примера ниже будем считать, что таблица называется **Users** и содержит:

| Поле | Назначение |
| --- | --- |
| **Id** | идентификатор пользователя |
| **Fio** | ФИО |
| **Login** | логин |
| **Password** | пароль |
| **Role** | роль |

Если в вашей базе таблица или поля называются иначе, **не нужно переделывать базу только ради этого примера**. Просто приведите модель **User** в соответствие с вашей базой.

Роли из задания:

- **Гость** — отдельный режим, пользователь в базе для него не нужен;
- **Авторизованный клиент**;
- **Менеджер**;
- **Администратор**.

## 2. Создаём Windows Forms проект

В Visual Studio:

1. Нажмите **Create a new project**.
2. Найдите шаблон **Windows Forms App**.
3. Выберите язык **C#**.
4. Не выбирайте шаблон **Windows Forms App (.NET Framework)**.
5. Выберите установленную версию современного .NET, которую используете на занятии/экзамене.
6. Создайте проект.

В этом примере назовём проект:

~~~text
DemoExam2026
~~~

Официальная инструкция Microsoft по созданию WinForms-проекта:

https://learn.microsoft.com/dotnet/desktop/winforms/get-started/create-app-visual-studio

## 3. Устанавливаем Entity Framework Core

Откройте:

**Project → Manage NuGet Packages**

Установите пакет:

**Microsoft.EntityFrameworkCore.SqlServer**

Либо через терминал:

~~~powershell
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
~~~

Пакет провайдера нужен для работы EF Core с SQL Server.

> Если в модуле 1 используется другая СУБД, выбирайте соответствующий EF Core provider. Например, для MySQL используется сторонний provider, а его версия должна соответствовать версии EF Core.

Для самой авторизации нам не нужны миграции: база уже создана в рамках ДЭ.

## 4. Создаём папки проекта

Чтобы код не лежал весь в **Form1.cs**, создайте:

~~~text
DemoExam2026
├── Data
├── Models
├── Services
└── Forms
~~~

В дальнейшем получится примерно такая структура:

~~~text
Data/
    AppDbContext.cs
Models/
    User.cs
Services/
    AuthService.cs
Forms/
    LoginForm.cs
    MainForm.cs
Program.cs
~~~

## 5. Создаём модель User

В папке **Models** создайте файл **User.cs**:

~~~csharp
namespace DemoExam2026.Models;

public enum UserRole
{
    AuthorizedClient,
    Manager,
    Administrator
}

public class User
{
    public int Id { get; set; }

    public string Fio { get; set; } = string.Empty;

    public string Login { get; set; } = string.Empty;

    public string Password { get; set; } = string.Empty;

    public UserRole Role { get; set; }
}
~~~

Это обычный C# класс.

EF Core использует его как модель сущности и связывает с таблицей базы данных.

## 6. Создаём DbContext

В папке **Data** создайте **AppDbContext.cs**:

~~~csharp
using DemoExam2026.Models;
using Microsoft.EntityFrameworkCore;

namespace DemoExam2026.Data;

public class AppDbContext : DbContext
{
    public DbSet<User> Users => Set<User>();

    protected override void OnConfiguring(
        DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(
            "Server=localhost;Database=DemoExam2026;Trusted_Connection=True;TrustServerCertificate=True;");
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>()
            .Property(user => user.Role)
            .HasConversion(
                role => role switch
                {
                    UserRole.AuthorizedClient => "Авторизованный клиент",
                    UserRole.Manager => "Менеджер",
                    UserRole.Administrator => "Администратор",
                    _ => throw new ArgumentOutOfRangeException()
                },
                value => value switch
                {
                    "Авторизованный клиент" => UserRole.AuthorizedClient,
                    "Менеджер" => UserRole.Manager,
                    "Администратор" => UserRole.Administrator,
                    _ => throw new ArgumentOutOfRangeException()
                });
    }
}
~~~

### Что здесь происходит

**DbContext** — основной класс EF Core для работы с базой.

`UserRole` — C#-перечисление с допустимыми ролями. В базе роль хранится как текстовое значение, поэтому настройка `HasConversion(...)` говорит EF Core, какое текстовое значение сохранять в столбец `Role` для каждой роли и как преобразовать его обратно в `UserRole`.

Строка:

~~~csharp
public DbSet<User> Users => Set<User>();
~~~

говорит, что через:

~~~csharp
db.Users
~~~

мы будем обращаться к таблице пользователей.

Строка:

~~~csharp
optionsBuilder.UseSqlServer(...)
~~~

настраивает подключение к SQL Server.

EF Core официально использует provider **Microsoft.EntityFrameworkCore.SqlServer** для SQL Server.

### Важный момент с connection string

Замените:

~~~text
Server=localhost;
Database=DemoExam2026;
Trusted_Connection=True;
TrustServerCertificate=True;
~~~

на параметры **своей** базы.

Например:

~~~text
Server=localhost\SQLEXPRESS;Database=DemoExam2026;Trusted_Connection=True;TrustServerCertificate=True;
~~~

Если база находится на другом компьютере:

~~~text
Server=192.168.1.10;Database=DemoExam2026;User Id=student;Password=your_password;TrustServerCertificate=True;
~~~

Не выкладывайте реальные пароли от базы в GitHub.

## 7. Создаём сервис авторизации

В папке **Services** создайте **AuthService.cs**:

~~~csharp
using DemoExam2026.Data;
using DemoExam2026.Models;

namespace DemoExam2026.Services;

public class AuthService
{
    public User? Authenticate(string login, string password)
    {
        using var db = new AppDbContext();

        return db.Users
            .FirstOrDefault(user =>
                user.Login == login &&
                user.Password == password);
    }
}
~~~

Теперь форма не должна сама заниматься запросами к базе.

Она передаёт логин и пароль в **AuthService**, а сервис возвращает:

- объект **User**, если пользователь найден;
- **null**, если пользователь не найден.

EF Core выполняет LINQ-запрос к **DbSet** и получает результат из базы.

## 8. Создаём окно авторизации

Переименуйте **Form1** в:

**LoginForm**.

Добавьте на форму:

| Элемент | Name | Text |
| --- | --- | --- |
| Label | — | Логин |
| TextBox | **txtLogin** | |
| Label | — | Пароль |
| TextBox | **txtPassword** | |
| Button | **btnLogin** | Войти |
| Button | **btnGuest** | Войти как гость |

Для **txtPassword** установите:

~~~text
UseSystemPasswordChar = true
~~~

## 9. Обрабатываем кнопку «Войти»

Дважды нажмите кнопку **Войти** в дизайнере.

В **LoginForm.cs**:

~~~csharp
using DemoExam2026.Models;
using DemoExam2026.Services;

namespace DemoExam2026.Forms;

public partial class LoginForm : Form
{
    private readonly AuthService _authService = new();

    public LoginForm()
    {
        InitializeComponent();
    }

    private void btnLogin_Click(object sender, EventArgs e)
    {
        string login = txtLogin.Text.Trim();
        string password = txtPassword.Text;

        if (string.IsNullOrWhiteSpace(login) ||
            string.IsNullOrWhiteSpace(password))
        {
            MessageBox.Show(
                "Введите логин и пароль.",
                "Ошибка",
                MessageBoxButtons.OK,
                MessageBoxIcon.Warning);

            return;
        }

        try
        {
            User? user = _authService.Authenticate(login, password);

            if (user == null)
            {
                MessageBox.Show(
                    "Неверный логин или пароль.",
                    "Ошибка авторизации",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Error);

                return;
            }

            OpenMainForm(user);
        }
        catch (Exception ex)
        {
            MessageBox.Show(
                $"Ошибка подключения к базе данных:\n{ex.Message}",
                "Ошибка",
                MessageBoxButtons.OK,
                MessageBoxIcon.Error);
        }
    }

    private void OpenMainForm(User user)
    {
        Hide();

        using var mainForm = new MainForm(user);

        mainForm.ShowDialog();

        Show();
    }
}
~~~

### Что происходит после нажатия кнопки

Алгоритм:

~~~text
Нажали «Войти»
        ↓
Получили логин и пароль
        ↓
Проверили, что поля заполнены
        ↓
Обратились к AuthService
        ↓
Пользователь найден?
     ↙       ↘
   нет        да
    ↓          ↓
 ошибка     открываем MainForm
               ↓
          передаём User
~~~

## 10. Добавляем гостевой режим

В **LoginForm.cs** создайте обработчик кнопки **btnGuest**:

~~~csharp
private void btnGuest_Click(object sender, EventArgs e)
{
    Hide();

    using var mainForm = new MainForm(null);

    mainForm.ShowDialog();

    Show();
}
~~~

Для гостя запись в таблице пользователей не нужна.

Мы просто передаём **null**.

## 11. Создаём MainForm

Добавьте новую форму:

**Add → Form (Windows Forms)**

Назовите её:

**MainForm.cs**.

Добавьте:

- Label **lblUser**;
- Button **btnLogout**;
- Panel или другие элементы будущего интерфейса товаров.

## 12. Передаём пользователя в MainForm

В **MainForm.cs**:

~~~csharp
using DemoExam2026.Models;

namespace DemoExam2026.Forms;

public partial class MainForm : Form
{
    private readonly User? _user;

    public MainForm(User? user)
    {
        InitializeComponent();

        _user = user;

        ConfigureInterface();
    }

    private void ConfigureInterface()
    {
        if (_user == null)
        {
            lblUser.Text = "Гость";
            return;
        }

        lblUser.Text = _user.Fio;

        switch (_user.Role)
        {
            case UserRole.AuthorizedClient:
                ConfigureClientInterface();
                break;

            case UserRole.Manager:
                ConfigureManagerInterface();
                break;

            case UserRole.Administrator:
                ConfigureAdminInterface();
                break;
        }
    }

    private void ConfigureClientInterface()
    {
        // Здесь оставляем только возможности клиента.
    }

    private void ConfigureManagerInterface()
    {
        // Здесь добавляем поиск, сортировку,
        // фильтрацию и просмотр заказов.
    }

    private void ConfigureAdminInterface()
    {
        // Здесь добавляем управление товарами
        // и управление заказами.
    }
}
~~~

Теперь приложение знает, кто вошёл.

Например:

~~~text
Авторизованный клиент
        ↓
только просмотр товаров

Менеджер
        ↓
просмотр + поиск + сортировка + фильтрация
+ просмотр заказов

Администратор
        ↓
возможности менеджера
+ управление товарами
+ управление заказами
~~~

Это соответствует распределению ролей в задании demo-exam-2026.

## 13. Добавляем выход

В обработчик кнопки **btnLogout**:

~~~csharp
private void btnLogout_Click(object sender, EventArgs e)
{
    Close();
}
~~~

Почему здесь достаточно **Close()**?

Потому что в **LoginForm** основной интерфейс был открыт через:

~~~csharp
using var mainForm = new MainForm(user);
mainForm.ShowDialog();
Show();
~~~

После закрытия **MainForm** выполнение возвращается в **LoginForm**, и вызывается:

~~~csharp
Show();
~~~

Поэтому пользователь снова видит окно авторизации.

## 14. Проверяем запуск приложения

Запустите приложение.

Проверьте четыре сценария.

### Сценарий 1. Неверный логин

Введите:

~~~text
login: test
password: test
~~~

Ожидаемый результат:

~~~text
Неверный логин или пароль.
~~~

### Сценарий 2. Правильный пользователь

Введите реальные данные пользователя из базы.

Ожидаемый результат:

1. окно авторизации скрывается;
2. открывается **MainForm**;
3. отображается ФИО;
4. интерфейс соответствует роли.

### Сценарий 3. Гость

Нажмите:

~~~text
Войти как гость
~~~

Ожидаемый результат:

~~~text
MainForm
Пользователь: Гость
~~~

### Сценарий 4. Выход

Нажмите:

~~~text
Выйти
~~~

Ожидаемый результат:

~~~text
MainForm закрывается
        ↓
LoginForm снова отображается
~~~

## 15. Типичные ошибки

### Ошибка 1. Неверное имя таблицы

Если база содержит:

~~~text
users
~~~

а в коде:

~~~csharp
modelBuilder.Entity<User>().ToTable("Users");
~~~

проверьте точное название таблицы для вашей СУБД.

### Ошибка 2. Неверное имя поля

Например, в базе:

~~~text
fio
~~~

а в модели ожидается:

~~~text
Fio
~~~

Приведите имя свойства модели в соответствие с реальным столбцом базы.

### Ошибка 3. Неправильная строка подключения

Сначала проверьте:

- адрес сервера;
- имя базы;
- способ авторизации;
- имя пользователя;
- пароль;
- доступность SQL Server.

### Ошибка 4. Поставили пакет другой версии EF Core

Provider EF Core должен соответствовать основной версии EF Core. Не следует смешивать provider одной major-версии с EF Core другой major-версии.

### Ошибка 5. Запрос находится прямо в кнопке

Не стоит превращать **btnLogin_Click** в огромный метод с подключением, SQL-запросом и обработкой всех данных.

Лучше:

~~~text
LoginForm
   ↓
AuthService
   ↓
AppDbContext
   ↓
Database
~~~

Так код проще читать и отлаживать.

## 16. Что нужно знать для ДЭ

Для авторизации достаточно понимать пять вещей:

1. **User** — модель пользователя.
2. **DbContext** — соединяет модель с базой.
3. **AuthService** — выполняет проверку логина и пароля.
4. **LoginForm** — получает данные от пользователя.
5. **MainForm** — получает авторизованного пользователя и показывает нужный интерфейс.

Главная схема:

~~~text
LoginForm
    │
    │ login + password
    ▼
AuthService
    │
    ▼
AppDbContext
    │
    ▼
Users
    │
    ├── пользователь не найден → ошибка
    │
    └── пользователь найден
             │
             ▼
          MainForm
             │
             └── Role → нужные возможности
~~~

## 17. Важное замечание про пароль

В примере пароль сравнивается напрямую, потому что он соответствует полю **Password** в учебных данных ДЭ.

Для реального приложения хранить пароли в открытом виде нельзя. В промышленном приложении пароль хранят как результат безопасного хеширования и сравнивают с хешем.

Для выполнения учебной работы не усложняйте авторизацию дополнительными механизмами, если они не требуются заданием.

## Полезные официальные материалы

- Windows Forms: https://learn.microsoft.com/dotnet/desktop/winforms/
- Создание WinForms-приложения: https://learn.microsoft.com/dotnet/desktop/winforms/get-started/create-app-visual-studio
- EF Core + Windows Forms: https://learn.microsoft.com/en-us/ef/core/get-started/winforms
- EF Core providers: https://learn.microsoft.com/en-us/ef/core/providers/
- EF Core SQL Server provider: https://learn.microsoft.com/en-us/ef/core/providers/sql-server/


---

# Частые вопросы

## Если вы видите EF Core впервые

Не пытайтесь запомнить весь код сразу.

Сначала нужно понять только одну цепочку:

~~~text
LoginForm
    ↓
AuthService
    ↓
AppDbContext
    ↓
EF Core
    ↓
Database
    ↓
User / null
    ↓
MainForm
    ↓
Role
~~~

У каждой части своя задача:

- **LoginForm** получает логин и пароль.
- **AuthService** проверяет пользователя.
- **AppDbContext** даёт приложению доступ к данным через EF Core.
- **User** описывает структуру пользователя.
- **MainForm** получает найденного пользователя и настраивает интерфейс.

Если эта схема понятна, остальной код уже не выглядит набором случайных строк.

---

## Почему User нужен, если пользователь уже есть в базе

База данных хранит строки:

~~~text
Id | Fio | Login | Password | Role
~~~

А C# удобнее работать с объектами:

~~~text
User
 ├── Id
 ├── Fio
 ├── Login
 ├── Password
 └── UserRole
~~~

EF Core связывает эти два представления:

~~~text
строка в БД
    ↕
EF Core
    ↕
объект User
~~~

Поэтому User.cs не создаёт вторую базу.

Это описание того, как представить запись из базы внутри C#.

---

## Что такое DbContext простыми словами

Не нужно думать:

~~~text
DbContext = база данных
~~~

Правильнее:

~~~text
DbContext = объект приложения для работы с базой
~~~

Он знает:

- какой provider использовать;
- куда подключаться;
- какие сущности существуют;
- как сущности сопоставляются с таблицами.

Когда мы создаём AppDbContext, мы создаём рабочий объект для выполнения запросов.

Сама база при этом существует отдельно.

---

## Что такое DbSet

Строка DbSet<User> Users создаёт точку доступа к сущности User.

Поэтому:

~~~text
db.Users
~~~

можно мысленно читать как:

~~~text
«работаем с пользователями»
~~~

Это не обычный список List<User>.

Когда мы строим запрос через db.Users, EF Core может сформировать SQL и выполнить его в базе.

---

## Что такое LINQ

LINQ позволяет писать запросы к данным с помощью C#.

Например:

~~~text
db.Users
    .Where(user => user.Role == "Менеджер")
~~~

мысленно читается:

~~~text
из Users выбрать пользователей,
у которых Role равен Менеджер
~~~

А запрос с FirstOrDefault можно читать как:

~~~text
найти первого пользователя,
который подходит под условие
~~~

EF Core передаёт представление LINQ-запроса провайдеру, который переводит его в язык конкретной базы данных.

---

## Что означает user => user.Login

Для новичка запись с user => может выглядеть странно.

Пока можно читать её так:

~~~text
для каждого пользователя
проверить его Login
и сравнить с введённым login
~~~

В авторизации проверяются два условия:

~~~text
логин совпадает
И
пароль совпадает
~~~

Оператор && означает логическое «И».

Поэтому неправильный пароль при правильном логине тоже даст результат null.

---

## Почему User? может быть null

Метод Authenticate возвращает User?.

Знак вопроса означает:

~~~text
результатом может быть User
или
null
~~~

Это соответствует реальной логике авторизации.

Если пользователь найден:

~~~text
User
~~~

Если пользователь не найден:

~~~text
null
~~~

Поэтому нельзя без проверки сразу обращаться к свойствам пользователя.

---

## Почему гость передаётся как null

Гость не является обычным пользователем из таблицы Users.

Поэтому:

~~~text
MainForm(null)
~~~

означает:

~~~text
авторизованного пользователя нет
~~~

В MainForm сначала проверяем отсутствие пользователя, показываем Гость и только потом работаем с ФИО и ролью обычного пользователя.

---

## Если структура БД отличается от модели

Если таблица или столбцы в вашей базе называются иначе, чем в классе `User`, EF Core нужно явно сообщить об этом.

Для этого используется настройка модели через `OnModelCreating`.

**В основном примере этот код не нужен**, потому что мы специально используем одинаковые названия `Users`, `Id`, `Fio`, `Login`, `Password`, `Role`.

Например, если в реальной базе таблица называется `user_accounts`, а столбец для логина — `user_login`, тогда уже понадобится дополнительная настройка соответствия.

---

## Почему сначала нужно посмотреть базу

Правильный порядок работы:

~~~text
1. Посмотреть БД
2. Узнать имя таблицы
3. Узнать имена столбцов
4. Узнать первичный ключ
5. Узнать типы данных
6. Создать User
7. Настроить DbContext
8. Написать AuthService
9. Подключить LoginForm
~~~

Не стоит сначала писать C# наугад, а потом пытаться заставить базу соответствовать коду.

В этой работе база уже существует, поэтому приложение подстраивается под неё.

---

## Почему не нужно создавать пользователей в User.cs

Не надо зашивать конкретного пользователя в модель:

~~~text
Login = admin
Password = 123
~~~

User.cs должен описывать структуру пользователя.

Реальные пользователи должны находиться в БД:

~~~text
User.cs
    ↓
описание полей

Database
    ↓
реальные пользователи
~~~

Если завтра в БД появится новый пользователь, код менять не потребуется.

---

## Почему запрос находится в AuthService

Технически можно было бы выполнять запрос прямо в обработчике кнопки.

Но тогда один метод отвечал бы сразу за:

~~~text
интерфейс
подключение
SQL
чтение результата
авторизацию
обработку ошибок
~~~

Лучше разделить ответственность:

~~~text
LoginForm
    ↓
AuthService
    ↓
AppDbContext
    ↓
Database
~~~

Так проще читать код, искать ошибки и расширять приложение.

---

## Что происходит после нажатия «Войти»

Полный процесс:

~~~text
1. Пользователь нажал кнопку
        ↓
2. LoginForm получает login и password
        ↓
3. Проверяются пустые поля
        ↓
4. Вызывается AuthService
        ↓
5. Создаётся DbContext
        ↓
6. Выполняется LINQ-запрос
        ↓
7. EF Core переводит запрос в SQL
        ↓
8. База ищет пользователя
        ↓
9. Возвращается User или null
        ↓
10. Если User найден — открывается MainForm
        ↓
11. MainForm получает Role
        ↓
12. Интерфейс настраивается под роль
~~~

Это главный алгоритм всей инструкции.

---

## Как искать ошибку, если авторизация не работает

Не меняйте сразу весь код.

Идите по цепочке:

~~~text
LoginForm
   ↓
получились ли login и password?
   ↓
AuthService
   ↓
создаётся ли AppDbContext?
   ↓
правильная ли connection string?
   ↓
правильная ли база?
   ↓
существует ли Users?
   ↓
совпадают ли названия столбцов?
   ↓
находит ли FirstOrDefault пользователя?
   ↓
правильная ли Role?
   ↓
открывается ли MainForm?
~~~

Если ошибка подключения — сначала проверяйте SQL Server и connection string.

Если ошибка связана с таблицей или столбцом — проверьте соответствие модели структуре БД.

Если пользователь не находится — проверяйте данные в БД.

Если пользователь найден, но интерфейс неправильный — проверяйте Role.

---

## Что делать, если роль не определяется

Проверьте точное значение Role в БД.

Например:

~~~text
Администратор
~~~

должно соответствовать значению, которое проверяется в switch.

Проблемой могут стать:

- другое написание;
- лишний пробел;
- другое значение роли;
- отличие регистра в конкретной СУБД.

Сначала посмотрите фактическое значение Role в таблице Users.

---

## Что делать, если таблица называется не Users

Если структура вашей базы отличается от примера, не нужно переделывать базу только ради инструкции.

Нужно настроить соответствие модели реальной структуре БД. Для этого в EF Core используется `OnModelCreating`.

Если структура совпадает с примером `Users(Id, Fio, Login, Password, Role)`, **никакой дополнительной настройки не требуется**.

---

## Что нужно понимать перед ДЭ

Не обязательно знать весь EF Core.

Минимум:

~~~text
Entity
↓
User

DbContext
↓
работа с БД

DbSet
↓
db.Users

LINQ
↓
запрос на C#

FirstOrDefault
↓
User или null

AuthService
↓
проверка

LoginForm
↓
ввод

MainForm
↓
интерфейс

UserUserRole
↓
права

null
↓
гость
~~~

И главная цепочка:

~~~text
LoginForm
    ↓
AuthService
    ↓
AppDbContext
    ↓
EF Core
    ↓
Database
    ↓
User / null
    ↓
MainForm
    ↓
Role
~~~

Если эту цепочку можете объяснить своими словами, основу авторизации на EF Core вы уже понимаете.

---

## Почему EF Core рекомендуется именно здесь

Авторизация — только первая часть приложения.

Дальше могут появиться:

~~~text
Users
Products
Orders
Categories
Suppliers
~~~

Для них можно использовать тот же подход:

~~~text
Product
Order
Category
Supplier
User
~~~

и:

~~~text
db.Products
db.Orders
db.Categories
db.Suppliers
db.Users
~~~

Поэтому EF Core удобно оставить основным способом работы с БД.

ADO.NET без ORM оставлен отдельно как учебный вариант, чтобы показать более низкоуровневую работу с подключением, SQL и результатом запроса.

---

## Что ответить преподавателю

Если спросят «как работает авторизация?»:

> Пользователь вводит логин и пароль в LoginForm. LoginForm передаёт их в AuthService. AuthService через AppDbContext выполняет LINQ-запрос к DbSet Users. EF Core переводит LINQ-запрос в SQL и получает данные из базы. Если пользователь найден, возвращается объект User. Он передаётся в MainForm, где по его Role настраивается интерфейс. Если выбран вход как гость, в MainForm передаётся null.

Если спросят «что такое ORM?»:

> ORM связывает объекты программы с таблицами базы данных и позволяет работать с данными через объекты и запросы C#.

Если спросят «почему EF Core?»:

> Он уменьшает количество ручного кода работы с БД, позволяет использовать C#-модели и LINQ и удобен при расширении приложения.
