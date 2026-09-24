# Авторизация в Windows Forms без ORM: ADO.NET

Этот вариант показывает, как сделать авторизацию для приложения из **demo-exam-2026** **без Entity Framework и другого ORM**.

Здесь мы напрямую работаем с базой через ADO.NET:

~~~text
Windows Forms
      ↓
AuthService
      ↓
SqlConnection
      ↓
SqlCommand
      ↓
SQL Server
~~~

В результате приложение будет:

- проверять логин и пароль по данным базы;
- определять роль пользователя;
- открывать основной интерфейс после успешного входа;
- показывать ФИО пользователя;
- позволять войти как гость;
- возвращать пользователя обратно на окно авторизации.

> Важно: в задании ДЭ база создаётся в модуле 1. Здесь мы **не создаём новую базу**, а подключаем уже существующую.

## 1. Что должно быть в базе

В исходных материалах ДЭ есть **user_import.xlsx** с пользователями.

Для примера будем считать, что после проектирования базы у нас есть таблица:

~~~text
Users
~~~

с полями:

| Поле | Назначение |
| --- | --- |
| **Id** | идентификатор |
| **Fio** | ФИО |
| **Login** | логин |
| **Password** | пароль |
| **Role** | роль |

Если у вас названия другие — используйте названия **своей** базы.

Роли из задания:

- **Авторизованный клиент**;
- **Менеджер**;
- **Администратор**.

Гость отдельной записью в базе не является.

## 2. Создаём Windows Forms проект

В Visual Studio:

1. Нажмите **Create a new project**.
2. Найдите **Windows Forms App**.
3. Выберите **C#**.
4. Не выбирайте **Windows Forms App (.NET Framework)**.
5. Выберите установленную версию современного .NET.
6. Создайте проект.

Назовём его:

~~~text
DemoExam2026
~~~

Официальная инструкция:

https://learn.microsoft.com/dotnet/desktop/winforms/get-started/create-app-visual-studio

## 3. Устанавливаем ADO.NET provider

Для SQL Server установите пакет:

**Microsoft.Data.SqlClient**

Через терминал:

~~~powershell
dotnet add package Microsoft.Data.SqlClient
~~~

В коде после этого используется:

~~~csharp
using Microsoft.Data.SqlClient;
~~~

Это современный ADO.NET provider для SQL Server. **Microsoft.Data.SqlClient** и старый **System.Data.SqlClient** — разные пакеты, их типы нельзя смешивать.

## 4. Создаём структуру проекта

Создайте:

~~~text
DemoExam2026
├── Models
├── Services
└── Forms
~~~

Получится:

~~~text
Models/
    User.cs
Services/
    AuthService.cs
Forms/
    LoginForm.cs
    MainForm.cs
Program.cs
~~~

Здесь нет:

~~~text
DbContext
DbSet
Entity Framework
~~~

Вместо этого будет обычный ADO.NET.

## 5. Создаём модель User

В папке **Models** создайте **User.cs**:

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

Модель здесь нужна только для удобной передачи данных пользователя из сервиса в форму.

Она **не является ORM-моделью**.

## 6. Создаём AuthService

В папке **Services** создайте:

**AuthService.cs**

~~~csharp
using DemoExam2026.Models;
using Microsoft.Data.SqlClient;

namespace DemoExam2026.Services;

public class AuthService
{
    private readonly string _connectionString =
        "Server=localhost;Database=DemoExam2026;Trusted_Connection=True;TrustServerCertificate=True;";

    public User? Authenticate(string login, string password)
    {
        const string sql = """
            SELECT Id, Fio, Login, Password, Role
            FROM Users
            WHERE Login = @Login
              AND Password = @Password
            """;

        using var connection = new SqlConnection(_connectionString);
        using var command = new SqlCommand(sql, connection);

        command.Parameters.AddWithValue("@Login", login);
        command.Parameters.AddWithValue("@Password", password);

        connection.Open();

        using var reader = command.ExecuteReader();

        if (!reader.Read())
        {
            return null;
        }

        return new User
        {
            Id = reader.GetInt32(reader.GetOrdinal("Id")),
            Fio = reader.GetString(reader.GetOrdinal("Fio")),
            Login = reader.GetString(reader.GetOrdinal("Login")),
            Password = reader.GetString(reader.GetOrdinal("Password")),
            Role = reader.GetString(reader.GetOrdinal("Role"))
        };
    }
}
~~~

## 7. Разбираем код AuthService

### Строка подключения

~~~csharp
private readonly string _connectionString =
    "Server=localhost;Database=DemoExam2026;Trusted_Connection=True;TrustServerCertificate=True;";
~~~

Здесь задаётся:

- сервер;
- база данных;
- способ подключения.

Замените параметры на свои.

Например:

~~~text
Server=localhost\SQLEXPRESS;
Database=DemoExam2026;
Trusted_Connection=True;
TrustServerCertificate=True;
~~~

Не записывайте реальные пароли доступа к БД в публичный репозиторий.

### SQL-запрос

~~~sql
SELECT Id, Fio, Login, Password, Role
FROM Users
WHERE Login = @Login
  AND Password = @Password
~~~

Он ищет пользователя, у которого совпали логин и пароль.

### Почему здесь используются @Login и @Password

Нельзя делать так:

~~~csharp
string sql =
    $"SELECT * FROM Users WHERE Login = '{login}' AND Password = '{password}'";
~~~

Это небезопасная конкатенация пользовательского ввода с SQL.

Нужно использовать параметры:

~~~csharp
command.Parameters.AddWithValue("@Login", login);
command.Parameters.AddWithValue("@Password", password);
~~~

Параметризованные команды позволяют отделить значения пользователя от текста SQL. Это важная защита от SQL-инъекций.

### Подключение к БД

~~~csharp
using var connection = new SqlConnection(_connectionString);
~~~

Создаётся подключение к SQL Server.

После:

~~~csharp
connection.Open();
~~~

подключение открывается.

### Выполнение запроса

~~~csharp
using var reader = command.ExecuteReader();
~~~

**SqlDataReader** позволяет читать строки результата запроса.

## 8. Создаём окно авторизации

Переименуйте **Form1** в:

**LoginForm**.

Добавьте:

| Элемент | Name | Text |
| --- | --- | --- |
| Label | — | Логин |
| TextBox | **txtLogin** | |
| Label | — | Пароль |
| TextBox | **txtPassword** | |
| Button | **btnLogin** | Войти |
| Button | **btnGuest** | Войти как гость |

Для поля пароля установите:

~~~text
UseSystemPasswordChar = true
~~~

## 9. Обрабатываем кнопку «Войти»

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

## 10. Добавляем гостя

В обработчике **btnGuest**:

~~~csharp
private void btnGuest_Click(object sender, EventArgs e)
{
    Hide();

    using var mainForm = new MainForm(null);

    mainForm.ShowDialog();

    Show();
}
~~~

Гость не проверяется через таблицу **Users**.

## 11. Создаём MainForm

Добавьте новую форму:

**Add → Form (Windows Forms)**

Назовите её:

**MainForm.cs**.

Добавьте:

- Label **lblUser**;
- Button **btnLogout**;
- элементы интерфейса товаров.

## 12. Передаём пользователя в MainForm

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
        // Только просмотр товаров.
    }

    private void ConfigureManagerInterface()
    {
        // Поиск, сортировка, фильтрация,
        // просмотр заказов.
    }

    private void ConfigureAdminInterface()
    {
        // Управление товарами
        // и заказами.
    }
}
~~~

## 13. Добавляем выход

В **MainForm.cs**:

~~~csharp
private void btnLogout_Click(object sender, EventArgs e)
{
    Close();
}
~~~

После закрытия **MainForm** выполнение возвращается в **LoginForm**, поэтому окно авторизации снова отображается.

## 14. Полный алгоритм

Получается такая схема:

~~~text
                   Запуск
                     ↓
               LoginForm
                 /     \
                /       \
             Гость      Войти
               ↓          ↓
           MainForm    проверка
                          ↓
                    AuthService
                          ↓
                    SQL Server
                       /     \
                      /       \
                  не найден   найден
                     ↓           ↓
                   ошибка       User
                                ↓
                            MainForm
                                ↓
                              Role
~~~

## 15. Проверяем авторизацию

### Проверка 1 — пустые поля

Нажмите **Войти**, ничего не вводя.

Должно появиться:

~~~text
Введите логин и пароль.
~~~

### Проверка 2 — неправильные данные

Введите несуществующий логин и пароль.

Должно появиться:

~~~text
Неверный логин или пароль.
~~~

### Проверка 3 — правильные данные

Введите пользователя из базы.

Должно произойти:

~~~text
LoginForm
    ↓
MainForm
~~~

На главной форме отображается ФИО.

### Проверка 4 — гость

Нажмите **Войти как гость**.

Должно отображаться:

~~~text
Гость
~~~

### Проверка 5 — выход

Нажмите **Выйти**.

Должно открыться окно авторизации.

## 16. Почему это называется «без ORM»

В варианте с Entity Framework мы пишем:

~~~csharp
db.Users.FirstOrDefault(...)
~~~

EF Core сам переводит LINQ-запрос в SQL.

Здесь такого слоя нет.

Мы сами пишем:

~~~sql
SELECT ...
FROM Users
WHERE ...
~~~

и сами создаём:

~~~text
SqlConnection
SqlCommand
SqlDataReader
~~~

Поэтому схема выглядит так.

### С ORM

~~~text
C# объект
   ↓
EF Core
   ↓
SQL
   ↓
База
~~~

### Без ORM

~~~text
C#
   ↓
ADO.NET
   ↓
SQL
   ↓
База
~~~

## 17. Типичные ошибки

### Ошибка 1. SQL вставлен через интерполяцию

Плохо:

~~~csharp
string sql =
    $"SELECT * FROM Users WHERE Login = '{login}'";
~~~

Хорошо:

~~~csharp
const string sql =
    "SELECT * FROM Users WHERE Login = @Login";

command.Parameters.AddWithValue("@Login", login);
~~~

Параметры нужны не только для красоты: они отделяют пользовательское значение от текста SQL и снижают риск SQL-инъекций.

### Ошибка 2. Используются неправильные namespace

Для современного SQL Server provider:

~~~csharp
using Microsoft.Data.SqlClient;
~~~

а не:

~~~csharp
using System.Data.SqlClient;
~~~

Эти типы относятся к разным пакетам и не являются взаимозаменяемыми.

### Ошибка 3. Неправильная строка подключения

Проверьте:

- сервер;
- базу;
- способ авторизации;
- пользователя;
- пароль;
- доступность SQL Server.

### Ошибка 4. Не открыли connection

Перед выполнением команды должно быть:

~~~csharp
connection.Open();
~~~

### Ошибка 5. Не закрываются ресурсы

Используйте:

~~~csharp
using var connection = new SqlConnection(...);
using var command = new SqlCommand(...);
using var reader = command.ExecuteReader();
~~~

Так ресурсы будут освобождаться автоматически.

## 18. Важное замечание про пароль

В примере пароль сравнивается с полем **Password**, потому что это соответствует учебным данным ДЭ.

Для реального приложения нельзя хранить пароли в открытом виде. В реальном проекте пароль должен храниться в виде безопасного хеша.

Для учебной реализации не добавляйте лишнюю сложность, если она не требуется конкретным заданием.

## 19. Что нужно знать для ДЭ

Для этого варианта достаточно понимать:

1. **User** — объект с данными пользователя.
2. **SqlConnection** — подключение к БД.
3. **SqlCommand** — выполнение SQL-команды.
4. **SqlDataReader** — чтение результата.
5. **AuthService** — отдельный класс для авторизации.
6. **LoginForm** — получает логин и пароль.
7. **MainForm** — получает пользователя и показывает интерфейс его роли.

Главная схема:

~~~text
LoginForm
    ↓
AuthService
    ↓
SqlConnection
    ↓
SqlCommand
    ↓
SQL Server
    ↓
SqlDataReader
    ↓
User
    ↓
MainForm
    ↓
Role
~~~

## Полезные официальные материалы

- Windows Forms: https://learn.microsoft.com/dotnet/desktop/winforms/
- Создание WinForms-приложения: https://learn.microsoft.com/dotnet/desktop/winforms/get-started/create-app-visual-studio
- Microsoft.Data.SqlClient: https://learn.microsoft.com/sql/connect/ado-net/introduction-microsoft-data-sqlclient-namespace
- SqlConnection: https://learn.microsoft.com/en-us/dotnet/api/microsoft.data.sqlclient.sqlconnection
- SqlCommand: https://learn.microsoft.com/en-us/dotnet/api/microsoft.data.sqlclient.sqlcommand
