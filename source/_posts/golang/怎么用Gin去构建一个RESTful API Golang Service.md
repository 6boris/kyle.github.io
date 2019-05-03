---
title: 怎么用Gin去构建一个RESTful API Golang Service【翻译】
date: 2018-09-07 09:25:00
author: Kyle Liu
<!-- img: /source/images/xxx.jpg -->
top: false
cover: false
<!-- coverImg: /images/1.jpg -->
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: true
mathjax: true
summary: 这是你自定义的文章摘要内容，如果这个属性有值，文章卡片摘要就显示这段文字，否则程序会自动截取文章的部分内容作为摘要
categories: Golang
tags:
  - Golang
  - Gin
---


# 怎么用Gin去构建一个RESTful API Golang Service【翻译】

##前言

[![Cover](http://oss.anonycurse.cn/article/images/20181105/531wGkj4qxe98UALsFO6vy9kRhdLLqVQZY0jnNhE.png "Cover")]("Cover")

Today I’m going to build a simple API for todo application with the golang programming language. I’m going to use golang simplest/fastest framework gin-gonic and a beautiful ORM gorm for our database work. To install these packages go to your workspace $GOPATH/src and run these command below:


```go
$ go get gopkg.in/gin-gonic/gin.v1
$ go get -u github.com/jinzhu/gorm
$ go get github.com/go-sql-driver/mysql
```
## In generic crud application we need the API’s as follows:
- POST todos/
- GET todos/
- GET todos/{id}
- PUT todos/{id}
- DELETE todos/{id}

Let’s start coding, go to your $GOPATH/src and make a directory todo. Inside the todo directory create a file main.go. Import the “gin framework” to our project and create the routes like below inside main function. I like to add a prefix of the apis like “api/v1/”, that’s why we’ll use the router Group method

```go
package main

import (
       "github.com/gin-gonic/gin"
)
func main() {
router := gin.Default()
v1 := router.Group("/api/v1/todos")
 {
  v1.POST("/", createTodo)
  v1.GET("/", fetchAllTodo)
  v1.GET("/:id", fetchSingleTodo)
  v1.PUT("/:id", updateTodo)
  v1.DELETE("/:id", deleteTodo)
 }
 router.Run()
}
```

We have created five routes and they handle some functions like `createTodo`, `fetchAllTodo` etc. We’ll discuss about them soon.

Now we need to setup a database connection. To use database pull the gorm package and mysql dialects in our code. Follow the code below:

```go
package main

import (
       "github.com/gin-gonic/gin"
       "github.com/jinzhu/gorm"
       _ "github.com/jinzhu/gorm/dialects/mysql"
)

var db *gorm.DB
func init() {
 //open a db connection
 var err error
 db, err = gorm.Open("mysql", "root:12345@/demo?charset=utf8&parseTime=True&loc=Local")
 if err != nil {
  panic("failed to connect database")
 }
//Migrate the schema
 db.AutoMigrate(&todoModel{})
}
```

In the above code “mysql” is our database driver, “root” is database username, “12345” password and “demo” is database name. Please change these information as your needs.

We’ll use the Database function to get the database connection. Lets make a todoModel and transformedTodo struct. The first struct will represent the original Todo and the second one will hold the transformed todo for response to the api. Here we transformed the todo response because we don’t expose some database fields (updated_at, created_at) to the consumer.

```go
type (
 // todoModel describes a todoModel type
 todoModel struct {
  gorm.Model
  Title     string `json:"title"`
  Completed int    `json:"completed"`
 }
// transformedTodo represents a formatted todo
 transformedTodo struct {
  ID        uint   `json:"id"`
  Title     string `json:"title"`
  Completed bool   `json:"completed"`
 }
)
```

Todo struct has one field extra gorm.Model what does it mean? well, this field will embed a Model struct for us which contains four fields “ID, CreatedAt, UpdatedAt, DeletedAt”

Gorm has migration facilities, we already used it in init function. When we run the application first it’ll create a connection and then the migration.
```go
//Migrate the schema
 db.AutoMigrate(&todoModel{})
```


![Migrate ](http://oss.anonycurse.cn/article/images/20181105/bhwTdjLFSN57B6kkl8CE8NVaZ7zeXGsYHhh7D0c0.png "Migrate ")


Can you remember the five routes we wrote a minute earlier? Lets implement the five methods one by one.

When a user send a POST request to the path ‘api/v1/todos/’ with ‘title and completed’ field it’ll be handled by this route v1.POST(“/”, createTodo)

Lets Implement the createTodo function

```go
// createTodo add a new todo
func createTodo(c *gin.Context) {
 completed, _ := strconv.Atoi(c.PostForm("completed"))
 todo := todoModel{Title: c.PostForm("title"), Completed: completed}
 db.Save(&todo)
 c.JSON(http.StatusCreated, gin.H{"status": http.StatusCreated, "message": "Todo item created successfully!", "resourceId": todo.ID})
}
```
In the above code we use gin Context to receive the posted data and gorm database connection to save the todo. After saving the resource we send the resource id with a good & meaningful response to the user.

Lets implement the rest of the functions

```go
// fetchAllTodo fetch all todos
func fetchAllTodo(c *gin.Context) {
 var todos []todoModel
 var _todos []transformedTodo
db.Find(&todos)
if len(todos) <= 0 {
  c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
  return
 }
//transforms the todos for building a good response
 for _, item := range todos {
  completed := false
  if item.Completed == 1 {
   completed = true
  } else {
   completed = false
  }
  _todos = append(_todos, transformedTodo{ID: item.ID, Title: item.Title, Completed: completed})
 }
 c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "data": _todos})
}
// fetchSingleTodo fetch a single todo
func fetchSingleTodo(c *gin.Context) {
 var todo todoModel
 todoID := c.Param("id")
db.First(&todo, todoID)
if todo.ID == 0 {
  c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
  return
 }
completed := false
 if todo.Completed == 1 {
  completed = true
 } else {
  completed = false
 }
_todo := transformedTodo{ID: todo.ID, Title: todo.Title, Completed: completed}
 c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "data": _todo})
}
// updateTodo update a todo
func updateTodo(c *gin.Context) {
 var todo todoModel
 todoID := c.Param("id")
db.First(&todo, todoID)
if todo.ID == 0 {
  c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
  return
 }
db.Model(&todo).Update("title", c.PostForm("title"))
 completed, _ := strconv.Atoi(c.PostForm("completed"))
 db.Model(&todo).Update("completed", completed)
 c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "message": "Todo updated successfully!"})
}
// deleteTodo remove a todo
func deleteTodo(c *gin.Context) {
 var todo todoModel
 todoID := c.Param("id")
db.First(&todo, todoID)
if todo.ID == 0 {
  c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
  return
 }
db.Delete(&todo)
 c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "message": "Todo deleted successfully!"})
}
```
In the fetchAllTodo function we fetched all the todos and and build a transformed response with id, title, completed . We removed the CreatedAt, UpdatedAt, DeletedAt fields and cast the integer value to bool.

Well, we write enough code, let try to build the app and test it, I’m going test it using chrome extension Postman (you can use any REST client like curl to test).

To build the app open your terminal and go the the project directory

```go
$ go build main.go
```

The command will build a binary file main and to run the file us this command $ ./main . Wow, our simple todo app is running on port: 8080. It’ll display the debug log, because by default gin run’s in debug mode and port 8080.

To test the api run postman and test the api sequentially

![Create a todo](http://oss.anonycurse.cn/article/images/20181105/wVld5PMtLBFEsVXjcE6pY9FhQSM5MuQB6hAJCtWe.png "Create a todo")

![Fetch all todos](http://oss.anonycurse.cn/article/images/20181105/KVYw3KtSm1CqkS1R9jaTdxBXwue69Rv68F0eD4gL.png "Fetch all todos")


![Fetch a single todo](http://oss.anonycurse.cn/article/images/20181105/Iye2ekzbzEPkO8ScOwqnV0tLSun07itdPsXannpm.png "Fetch a single todo")

![Update a todo](http://oss.anonycurse.cn/article/images/20181105/Y5VFuzWRyiocmO6JERjrZw5hvn9B7OBiQbAi4U0S.png "Update a todo")

![Delete a todo](http://oss.anonycurse.cn/article/images/20181105/8b3CN4i3tfZbtsOvFglwi3DLtpHCPlmwfHuuBsWb.png "Delete a todo")

## Need full source code?
```go
package main

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mysql"
)

var db *gorm.DB

func init() {
	//open a db connection
	var err error
	db, err = gorm.Open("mysql", "root:12345@/demo?charset=utf8&parseTime=True&loc=Local")
	if err != nil {
		panic("failed to connect database")
	}

	//Migrate the schema
	db.AutoMigrate(&todoModel{})
}

func main() {

	router := gin.Default()

	v1 := router.Group("/api/v1/todos")
	{
		v1.POST("/", createTodo)
		v1.GET("/", fetchAllTodo)
		v1.GET("/:id", fetchSingleTodo)
		v1.PUT("/:id", updateTodo)
		v1.DELETE("/:id", deleteTodo)
	}
	router.Run()

}

type (
	// todoModel describes a todoModel type
	todoModel struct {
		gorm.Model
		Title     string `json:"title"`
		Completed int    `json:"completed"`
	}

	// transformedTodo represents a formatted todo
	transformedTodo struct {
		ID        uint   `json:"id"`
		Title     string `json:"title"`
		Completed bool   `json:"completed"`
	}
)

// createTodo add a new todo
func createTodo(c *gin.Context) {
	completed, _ := strconv.Atoi(c.PostForm("completed"))
	todo := todoModel{Title: c.PostForm("title"), Completed: completed}
	db.Save(&todo)
	c.JSON(http.StatusCreated, gin.H{"status": http.StatusCreated, "message": "Todo item created successfully!", "resourceId": todo.ID})
}

// fetchAllTodo fetch all todos
func fetchAllTodo(c *gin.Context) {
	var todos []todoModel
	var _todos []transformedTodo

	db.Find(&todos)

	if len(todos) <= 0 {
		c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
		return
	}

	//transforms the todos for building a good response
	for _, item := range todos {
		completed := false
		if item.Completed == 1 {
			completed = true
		} else {
			completed = false
		}
		_todos = append(_todos, transformedTodo{ID: item.ID, Title: item.Title, Completed: completed})
	}
	c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "data": _todos})
}

// fetchSingleTodo fetch a single todo
func fetchSingleTodo(c *gin.Context) {
	var todo todoModel
	todoID := c.Param("id")

	db.First(&todo, todoID)

	if todo.ID == 0 {
		c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
		return
	}

	completed := false
	if todo.Completed == 1 {
		completed = true
	} else {
		completed = false
	}

	_todo := transformedTodo{ID: todo.ID, Title: todo.Title, Completed: completed}
	c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "data": _todo})
}

// updateTodo update a todo
func updateTodo(c *gin.Context) {
	var todo todoModel
	todoID := c.Param("id")

	db.First(&todo, todoID)

	if todo.ID == 0 {
		c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
		return
	}

	db.Model(&todo).Update("title", c.PostForm("title"))
	completed, _ := strconv.Atoi(c.PostForm("completed"))
	db.Model(&todo).Update("completed", completed)
	c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "message": "Todo updated successfully!"})
}

// deleteTodo remove a todo
func deleteTodo(c *gin.Context) {
	var todo todoModel
	todoID := c.Param("id")

	db.First(&todo, todoID)

	if todo.ID == 0 {
		c.JSON(http.StatusNotFound, gin.H{"status": http.StatusNotFound, "message": "No todo found!"})
		return
	}

	db.Delete(&todo)
	c.JSON(http.StatusOK, gin.H{"status": http.StatusOK, "message": "Todo deleted successfully!"})
}
```

> Note: When you are using code production you must take care of the steps below:

- Do not fetch all the data select * from todos , use pagination
- Do not trust user input. You must validate the inputs, there are severals tools to validate input. Read the article for validation process
- Check every possible error
- You should use logging and authentication as your need
- I am really sorry for my BAD English and Writing flow. If you notice any mistake in the article please feel free to write a comment.