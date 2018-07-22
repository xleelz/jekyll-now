---
layout: post
title: Using MSSQL in C# Part 1 - Basic Connection
---

## Spoilers

This tutorial is meant for people with prior database knowledge and the basic knowledge that languages connecting to a database through a database driver.

---

So, I've recently started learning C# and .NET development. After the very basics, of course, the first thing I really wanted to know was how to fetch data from a database.

It turns out, C# has a default namespace ```csharp System.Data.SqlClient``` that handles connecting to sql.
<br><br>

Not a lot of C# tutorials include the boilerplate required to start the program. I'm a beginner, so it's good practice for me:

{% highlight csharp %}
using System;
using System.Data.SqlClient;

namespace Application {

    class MainClass {

        static int Main(string[] args) {

        }
    }
}
{% endhighlight %}

This should get us started.

>Note: This application does absolutey nothing up to this point.

## Connection String

Now, we need a ***connection string***. 


A Connection string is a string used to connect to a database through a database driver. It often can tell the type of driver needed (based on if you are connecting to MySQL, MSSQL, etc.). The connection string can also tell what server the database is on, and the user/password combo to login to the database.


A typical connection string in C# will look something like this: 
{% highlight csharp %}
string connString = @"Data Source=[Server URL]; 
                        Initial Catalog=[Database name]; 
                        Integrated Security=[Integrated Security]; 
                        User ID=[Your Database User ID]; 
                        Password=[Your Database Password]";
{% endhighlight %}

- **Data Source:** The url or ip address of the database server you are trying to connect to
- **Initial Catalog:** The name of the database you are trying to connect to
- **Integrated Security:** Flag for if you want to include default windows security.
    - ```true``` or ```yes``` or ```sspi``` - if you want to use default windows account security
    - ```false``` or ```no``` - if you would rather use the database credentials for authentication
- **User ID:** your database user id (not included if Integrated Security is true, yes, or sspi)
- **Password:** your database password (not included if Integrated Security is true, yes, or sspi)
<br>


Now, We can add a ```using``` statement to our code and use the ```SqlConnect``` and ```SqlCommand``` classes:
{% highlight csharp %}
using System;
using System.Data;
using System.Data.SqlClient;

namespace Application {

    class MainClass {

        static int Main(string[] args) {

            //Uses the using statement because SqlConnect is a disposable class
            //SqlConnect takes a connection string as it's parameter for connecting
            // to a databse
            using(SqlConnect conn = new SqlConnect(@"Data Source=localhost;
                                                        Initial Catalog=test;
                                                        Integrated Security=false;
                                                        User ID=user;
                                                        Passowrd=password")) {

                //SqlCommand takes a sql query as a first parameter and a
                // SqlConnect object as a second parameter and runs the query
                // on the given database connection
                SqlCommand cmd = new SqlCommand("SELECT * FROM Users;", conn);


                //SqlDataReader class holds all the functionality to
                // store and retrieve results from a SqlCommand statement
                //The cmd.ExecuteReader method fetches the results from the
                // given sql statement
                SqlDataReader reader = cmd.ExecuteReader();

                //Loop and print all results to the console
                while (reader.Read()) {

                    //SqlDataReader objects hold columns of the result set
                    // as their indexes. So the UserId (column 0) is being printed
                    Console.WriteLine(reader[0]);
                }
            }
        }
    }
}
{% endhighlight %}

## Summary

1. The ```SqlConnect``` object takes a connection string to connect to a database. 
1. Then, the ```SqlCommand``` object
takes the query you would like to execute on the database, as well as the connection object. 
1. Lastly, ```SqlDataReader``` holds the results of the query. In each iteration of ```SqlDataReader```, the columns of the
result set are the indexes of the array.

I hope this has been at least a little helpful to you.