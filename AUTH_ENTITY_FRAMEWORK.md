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

Если в вашей базе таблица или поля называются иначе, **не нужно переделывать базу только ради этого примера**. Просто замените имена в классе **User** и настройке **OnModelCreating**.

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

public class User
{
    public int Id { get; set; }

    public string Fio { get; set; } = string.Empty;

    public string Login { get; set; } = string.Empty;

    public string Password { get; set; } = string.Empty;

    public string Role { get; set; } = string.Empty;
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
        modelBuilder.Entity<User>().ToTable("Users");

        modelBuilder.Entity<User>()
            .HasKey(user => user.Id);

        modelBuilder.Entity<User>()
            .Property(user => user.Fio)
            .HasColumnName("Fio");

        modelBuilder.Entity<User>()
            .Property(user => user.Login)
            .HasColumnName("Login");

        modelBuilder.Entity<User>()
            .Property(user => user.Password)
            .HasColumnName("Password");

        modelBuilder.Entity<User>()
            .Property(user => user.Role)
            .HasColumnName("Role");
    }
}
~~~

### Что здесь происходит

**DbContext** — основной класс EF Core для работы с базой.

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
            case "Авторизованный клиент":
                ConfigureClientInterface();
                break;

            case "Менеджер":
                ConfigureManagerInterface();
                break;

            case "Администратор":
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

Настройте соответствие через **HasColumnName**.

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
