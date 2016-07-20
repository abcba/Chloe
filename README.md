# Chloe
Chloe is a lightweight Object/Relational Mapping(ORM) library.
# Usage
* Entity
```C#
public enum Gender
{
    Man = 1,
    Woman
}

[Table("Users")]
public class User
{
    [Column(IsPrimaryKey = true)]
    [AutoIncrement]
    public int Id { get; set; }
    public string Name { get; set; }
    public Gender? Gender { get; set; }
    public int? Age { get; set; }
    public int? CityId { get; set; }
    public DateTime? OpTime { get; set; }
}

public class City
{
    [Column(IsPrimaryKey = true)]
    public int Id { get; set; }
    public string Name { get; set; }
    public int ProvinceId { get; set; }
}

public class Province
{
    [Column(IsPrimaryKey = true)]
    public int Id { get; set; }
    public string Name { get; set; }
}
```
* DbContext
```C#
IDbContext context = new MsSqlContext(DbHelper.ConnectionString);
IQuery<User> q = context.Query<User>();
```
* Query
```C#
IQuery<User> q = context.Query<User>();
q.Where(a => a.Id > 0).FirstOrDefault();
q.Where(a => a.Id > 0).ToList();
q.Where(a => a.Id > 0).OrderBy(a => a.Age).ToList();
q.Where(a => a.Id > 0).Take(999).OrderBy(a => a.Age).ToList();

q.Where(a => a.Id > 0).OrderBy(a => a.Age).ThenByDesc(a => a.Id).Select(a => new { a.Id, a.Name }).Skip(1).Take(999).ToList();
/*
 * SELECT TOP (999) [T].[Id] AS [Id],[T].[Name] AS [Name] FROM (SELECT [Users].[Id] AS [Id],[Users].[Name] AS [Name],ROW_NUMBER() OVER(ORDER BY [Users].[Age] ASC,[Users].[Id] DESC) AS [ROW_NUMBER_0] FROM [Users] AS [Users] WHERE [Users].[Id] > 0) AS [T] WHERE [T].[ROW_NUMBER_0] > 1
 */

q.Where(a => a.Id > 0).Where(a => a.Name.Contains("lu")).ToList();
/*
 * SELECT [Users].[Id] AS [Id],[Users].[Name] AS [Name],[Users].[Gender] AS [Gender],[Users].[Age] AS [Age],[Users].[CityId] AS [CityId],[Users].[OpTime] AS [OpTime] 
 * FROM [Users] AS [Users] 
 * WHERE ([Users].[Id] > 0 AND [Users].[Name] LIKE '%' + N'lu' + '%')
 */
```
* Join Query
```C#
MsSqlContext context = new MsSqlContext(DbHelper.ConnectionString);
IQuery<User> users = context.Query<User>();
IQuery<City> cities = context.Query<City>();
IQuery<Province> provinces = context.Query<Province>();

IJoiningQuery<User, City> user_city = users.InnerJoin(cities, (user, city) => user.CityId == city.Id);
IJoiningQuery<User, City, Province> user_city_province = user_city.InnerJoin(provinces, (user, city, province) => city.ProvinceId == province.Id);

user_city_province.Select((user, city, province) => new { UserId = user.Id, CityName = city.Name, ProvinceName = province.Name }).Where(a => a.UserId == 1).ToList();
/*
 * SELECT [Users].[Id] AS [UserId],[City].[Name] AS [CityName],[Province].[Name] AS [ProvinceName] 
 * FROM [Users] AS [Users] 
 * INNER JOIN [City] AS [City] ON [Users].[CityId] = [City].[Id] 
 * INNER JOIN [Province] AS [Province] ON [City].[ProvinceId] = [Province].[Id] 
 * WHERE [Users].[Id] = 1
 */
 
var view = user_city_province.Select((user, city, province) => new { User = user, City = city, Province = province });
 
view.Where(a => a.User.Id == 1).ToList();
/*
 * SELECT [Users].[Id] AS [Id],[Users].[Name] AS [Name],[Users].[Gender] AS [Gender],[Users].[Age] AS [Age],[Users].[CityId] AS [CityId],[Users].[OpTime] AS [OpTime],[City].[Id] AS [Id0],[City].[Name] AS [Name0],[City].[ProvinceId] AS [ProvinceId],[Province].[Id] AS [Id1],[Province].[Name] AS [Name1] 
 * FROM [Users] AS [Users] 
 * INNER JOIN [City] AS [City] ON [Users].[CityId] = [City].[Id] 
 * INNER JOIN [Province] AS [Province] ON [City].[ProvinceId] = [Province].[Id] 
 * WHERE [Users].[Id] = 1
 */
 
view.Where(a => a.User.Id == 1).Select(a => new { UserId = a.User.Id, CityName = a.City.Name, ProvinceName = a.Province.Name }).ToList();
/*
 * SELECT [Users].[Id] AS [UserId],[City].[Name] AS [CityName],[Province].[Name] AS [ProvinceName] 
 * FROM [Users] AS [Users] 
 * INNER JOIN [City] AS [City] ON [Users].[CityId] = [City].[Id] 
 * INNER JOIN [Province] AS [Province] ON [City].[ProvinceId] = [Province].[Id] 
 * WHERE [Users].[Id] = 1
 */

/*
 * Chloe also supports left join,right join and full join query.
 * For details please see 'https://github.com/shuxinqin/Chloe/blob/master/Chloe/IQuery%60.cs'.
 */
```
* Group Query
```C#
IQuery<User> q = context.Query<User>();

IGroupingQuery<User> g = q.Where(a => a.Id > 0).GroupBy(a => a.Age);
g = g.Having(a => a.Age > 1 && AggregateFunctions.Count() > 0);

g.Select(a => new { a.Age, Count = AggregateFunctions.Count(), Sum = AggregateFunctions.Sum(a.Age), Max = AggregateFunctions.Max(a.Age), Min = AggregateFunctions.Min(a.Age), Avg = AggregateFunctions.Average(a.Age) }).ToList();
/*
 * SELECT [Users].[Age] AS [Age],COUNT(1) AS [Count],SUM([Users].[Age]) AS [Sum],MAX([Users].[Age]) AS [Max],MIN([Users].[Age]) AS [Min],CAST(AVG([Users].[Age]) AS FLOAT) AS [Avg] 
 * FROM [Users] AS [Users] 
 * WHERE [Users].[Id] > 0 
 * GROUP BY [Users].[Age] HAVING ([Users].[Age] > 1 AND COUNT(1) > 0)
 */
```
* Sql Query
```C#
context.SqlQuery<User>("select Id,Name,Age from Users where Name=@name", DbParam.Create("@name", "lu")).ToList();
context.SqlQuery<int>("select Id from Users").ToList();
```
* Aggregate Function
```C#
IQuery<User> q = context.Query<User>();

q.Select(a => AggregateFunctions.Count()).First();
/*
 * SELECT TOP (1) COUNT(1) AS [C] FROM [Users] AS [Users]
 */

q.Select(a => new { Count = AggregateFunctions.Count(), LongCount = AggregateFunctions.LongCount(), Sum = AggregateFunctions.Sum(a.Age), Max = AggregateFunctions.Max(a.Age), Min = AggregateFunctions.Min(a.Age), Average = AggregateFunctions.Average(a.Age) }).First();
/*
 * SELECT TOP (1) COUNT(1) AS [Count],COUNT_BIG(1) AS [LongCount],SUM([Users].[Age]) AS [Sum],MAX([Users].[Age]) AS [Max],MIN([Users].[Age]) AS [Min],CAST(AVG([Users].[Age]) AS FLOAT) AS [Average] 
 * FROM [Users] AS [Users]
 */

var count = q.Count();
/*
 * SELECT COUNT(1) AS [C] FROM [Users] AS [Users]
 */

var longCount = q.LongCount();
/*
 * SELECT COUNT_BIG(1) AS [C] FROM [Users] AS [Users]
 */

var sum = q.Sum(a => a.Age);
/*
 * SELECT SUM([Users].[Age]) AS [C] FROM [Users] AS [Users]
 */

var max = q.Max(a => a.Age);
/*
 * SELECT MAX([Users].[Age]) AS [C] FROM [Users] AS [Users]
 */

var min = q.Min(a => a.Age);
/*
 * SELECT MIN([Users].[Age]) AS [C] FROM [Users] AS [Users]
 */

var avg = q.Average(a => a.Age);
/*
 * SELECT CAST(AVG([Users].[Age]) AS FLOAT) AS [C] FROM [Users] AS [Users]
 */
```
* Method
```C#
IQuery<User> q = context.Query<User>();

var space = new char[] { ' ' };

DateTime startTime = DateTime.Now;
DateTime endTime = DateTime.Now.AddDays(1);
q.Select(a => new
{
    Id = a.Id,

    String_Length = (int?)a.Name.Length,//LEN([Users].[Name])
    Substring = a.Name.Substring(0),//SUBSTRING([Users].[Name],0 + 1,LEN([Users].[Name]))
    Substring1 = a.Name.Substring(1),//SUBSTRING([Users].[Name],1 + 1,LEN([Users].[Name]))
    Substring1_2 = a.Name.Substring(1, 2),//SUBSTRING([Users].[Name],1 + 1,2)
    ToLower = a.Name.ToLower(),//LOWER([Users].[Name])
    ToUpper = a.Name.ToUpper(),//UPPER([Users].[Name])
    IsNullOrEmpty = string.IsNullOrEmpty(a.Name),//too long
    Contains = (bool?)a.Name.Contains("s"),//
    Trim = a.Name.Trim(),//RTRIM(LTRIM([Users].[Name]))
    TrimStart = a.Name.TrimStart(space),//LTRIM([Users].[Name])
    TrimEnd = a.Name.TrimEnd(space),//RTRIM([Users].[Name])
    StartsWith = (bool?)a.Name.StartsWith("s"),//
    EndsWith = (bool?)a.Name.EndsWith("s"),//

    SubtractTotalDays = endTime.Subtract(startTime).TotalDays,//CAST(DATEDIFF(DAY,@P_0,@P_1)
    SubtractTotalHours = endTime.Subtract(startTime).TotalHours,//CAST(DATEDIFF(HOUR,@P_0,@P_1)
    SubtractTotalMinutes = endTime.Subtract(startTime).TotalMinutes,//CAST(DATEDIFF(MINUTE,@P_0,@P_1)
    SubtractTotalSeconds = endTime.Subtract(startTime).TotalSeconds,//CAST(DATEDIFF(SECOND,@P_0,@P_1)
    SubtractTotalMilliseconds = endTime.Subtract(startTime).TotalMilliseconds,//CAST(DATEDIFF(MILLISECOND,@P_0,@P_1)

    Now = DateTime.Now,//GETDATE()
    UtcNow = DateTime.UtcNow,//GETUTCDATE()
    Today = DateTime.Today,//CAST(GETDATE() AS DATE)
    Date = DateTime.Now.Date,//CAST(GETDATE() AS DATE)
    Year = DateTime.Now.Year,//DATEPART(YEAR,GETDATE())
    Month = DateTime.Now.Month,//DATEPART(MONTH,GETDATE())
    Day = DateTime.Now.Day,//DATEPART(DAY,GETDATE())
    Hour = DateTime.Now.Hour,//DATEPART(HOUR,GETDATE())
    Minute = DateTime.Now.Minute,//DATEPART(MINUTE,GETDATE())
    Second = DateTime.Now.Second,//DATEPART(SECOND,GETDATE())
    Millisecond = DateTime.Now.Millisecond,//DATEPART(MILLISECOND,GETDATE())
    DayOfWeek = DateTime.Now.DayOfWeek,//(DATEPART(WEEKDAY,GETDATE()) - 1)

    Int_Parse = int.Parse("1"),//CAST(N'1' AS INT)
    Int16_Parse = Int16.Parse("11"),//CAST(N'11' AS SMALLINT)
    Long_Parse = long.Parse("2"),//CAST(N'2' AS BIGINT)
    Double_Parse = double.Parse("3"),//CAST(N'3' AS FLOAT)
    Float_Parse = float.Parse("4"),//CAST(N'4' AS REAL)
    //Decimal_Parse = decimal.Parse("5"),//CAST(N'5' AS DECIMAL)  ps: 'Decimal.Parse(string s)' is not supported now,because we don't know the precision and scale information.
    Guid_Parse = Guid.Parse("D544BC4C-739E-4CD3-A3D3-7BF803FCE179"),//CAST(N'D544BC4C-739E-4CD3-A3D3-7BF803FCE179' AS UNIQUEIDENTIFIER) AS [Guid_Parse]

    Bool_Parse = bool.Parse("1"),//CASE WHEN CAST(N'1' AS BIT) = CAST(1 AS BIT) THEN CAST(1 AS BIT) WHEN NOT (CAST(N'1' AS BIT) = CAST(1 AS BIT)) THEN CAST(0 AS BIT) ELSE NULL END AS [Bool_Parse]
    DateTime_Parse = DateTime.Parse("1992-1-16"),//CAST(N'1992-1-16' AS DATETIME) AS [DateTime_Parse]

    B = a.Age == null ? false : a.Age > 1,
}).ToList();
```
* Insert
```C#
IDbContext context = new MsSqlContext(DbHelper.ConnectionString);

//return the key value
int id = (int)context.Insert<User>(() => new User() { Name = "lu", Age = 18, Gender = Gender.Man, CityId = 1, OpTime = DateTime.Now });
/*
 * INSERT INTO [Users]([Name],[Age],[Gender],[CityId],[OpTime]) VALUES(N'lu',18,1,1,GETDATE());SELECT @@IDENTITY
 */

 
User user = new User();
user.Name = "lu";
user.Age = 18;
user.Gender = Gender.Man;
user.CityId = 1;
user.OpTime = new DateTime(1992, 1, 16);

user = context.Insert(user);
/*
 * String @P_0 = "lu";
   Gender @P_1 = Man;
   Int32 @P_2 = 18;
   Int32 @P_3 = 1;
   DateTime @P_4 = "1992/1/16 0:00:00";
   INSERT INTO [Users]([Name],[Gender],[Age],[CityId],[OpTime]) VALUES(@P_0,@P_1,@P_2,@P_3,@P_4);SELECT @@IDENTITY
 */
```
* Update
```C#
MsSqlContext context = new MsSqlContext(DbHelper.ConnectionString);

context.Update<User>(a => new User() { Name = a.Name, Age = a.Age + 100, Gender = Gender.Man, OpTime = DateTime.Now }, a => a.Id == 1);
/*
 * UPDATE [Users] SET [Name]=[Users].[Name],[Age]=([Users].[Age] + 100),[Gender]=1,[OpTime]=GETDATE() WHERE [Users].[Id] = 1
 */

context.Update<User>(a => new User() { Age = a.Age - 10, OpTime = DateTime.Now }, a => a.Gender == Gender.Woman);
/*
 * UPDATE [Users] SET [Age]=([Users].[Age] - 10),[OpTime]=GETDATE() WHERE [Users].[Gender] = 2
 */


User user = new User();
user.Id = 1;
user.Name = "lu";
user.Age = 28;
user.Gender = Gender.Man;
user.OpTime = DateTime.Now;

context.Update(user); //update all columns
/*
 * String @P_0 = "lu";
   Gender @P_1 = Man;
   Int32 @P_2 = 28;
   Nullable<Int32> @P_3 = NULL;
   DateTime @P_4 = "2016/7/8 11:28:27";
   Int32 @P_5 = 1;
   UPDATE [Users] SET [Name]=@P_0,[Gender]=@P_1,[Age]=@P_2,[CityId]=@P_3,[OpTime]=@P_4 WHERE [Users].[Id] = @P_5
 */

context.TrackEntity(user);//track entity
user.Name = user.Name + "1";
context.Update(user);//update the column 'Name' only
/*
 * String @P_0 = "lu1";
   Int32 @P_1 = 1;
   UPDATE [Users] SET [Name]=@P_0 WHERE [Users].[Id] = @P_1
 */
```
* Delete
```C#
MsSqlContext context = new MsSqlContext(DbHelper.ConnectionString);

context.Delete<User>(a => a.Id == 1);
/*
 * DELETE [Users] WHERE [Users].[Id] = 1
 */

context.Delete<User>(a => a.Gender == null);
/*
 * DELETE [Users] WHERE [Users].[Gender] IS NULL
 */
 
 
User user = new User();
user.Id = 1;
context.Delete(user);
/*
 * Int32 @P_0 = 1;
   DELETE [Users] WHERE [Users].[Id] = @P_0
 */
```

[More...](http://www.cnblogs.com/so9527/p/5636216.html)
