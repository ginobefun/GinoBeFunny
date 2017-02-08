---
title: Protocol Buffers简明教程
date: 2017-02-07 19:32:47
tags: [Protocol Buffers, protobuf, Google]
categories: OpenSource
link_title: learning_protobuf
---
[Protocol Buffers](https://github.com/google/protobuf)是Google开源的一个语言无关、平台无关的通信协议，其小巧、高效和友好的兼容性设计，使其被广泛使用。
<!-- more -->

## 概述
### protobuf是什么？
> Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages.

### 特点
- 语言无关、平台无关
- 简洁
- 高性能
- 良好的兼容性

## 快速开始
以下示例源码已上传至github：https://github.com/ginobefun/learning_projects/tree/master/learning-protobuf

### 新建一个maven项目并添加依赖

    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
    
        <groupId>com.ginobefunny.learning</groupId>
        <artifactId>leanring-protobuf</artifactId>
        <version>1.0-SNAPSHOT</version>
    
        <dependencies>
            <dependency>
                <groupId>com.google.protobuf</groupId>
                <artifactId>protobuf-java</artifactId>
                <version>3.2.0</version>
            </dependency>
        </dependencies>
    </project>

### 新建protobuf的消息定义文件addressbook.proto

    syntax = "proto3"; // 声明为protobuf 3定义文件
    package tutorial;
    
    option java_package = "com.ginobefunny.learning.protobuf.message"; // 声明生成消息类的java包路径
    option java_outer_classname = "AddressBookProtos";  // 声明生成消息类的类名
    
    message Person {
      string name = 1;
      int32 id = 2;
      string email = 3;
    
      enum PhoneType {
        MOBILE = 0;
        HOME = 1;
        WORK = 2;
      }
    
      message PhoneNumber {
        string number = 1;
        PhoneType type = 2;
      }
    
      repeated PhoneNumber phones = 4;
    }
    
    message AddressBook {
      repeated Person people = 1;
    }

### 使用protoc工具生成消息对应的Java类

- 从[已发布版本](https://github.com/google/protobuf/releases/)中下载protoc工具，比如protoc-3.2.0-win32；
- 解压后将bin目录添加到path路径；
- 执行以下protoc命令生成Java类：


    protoc -I=. --java_out=src/main/java addressbook.proto


### 编写测试类写入和读取序列化文件
- AddPerson类通过用户每次添加一个联系人，并序列化保存到指定文件中。

```java
public class AddPerson {

    // 通过用户输入构建一个Person对象
    static AddressBookProtos.Person promptForAddress(BufferedReader stdin,
                                                     PrintStream stdout) throws IOException {
        AddressBookProtos.Person.Builder person = AddressBookProtos.Person.newBuilder();

        stdout.print("Enter person ID: ");
        person.setId(Integer.valueOf(stdin.readLine()));

        stdout.print("Enter name: ");
        person.setName(stdin.readLine());

        stdout.print("Enter email address (blank for none): ");
        String email = stdin.readLine();
        if (email.length() > 0) {
            person.setEmail(email);
        }

        while (true) {
            stdout.print("Enter a phone number (or leave blank to finish): ");
            String number = stdin.readLine();
            if (number.length() == 0) {
                break;
            }

            AddressBookProtos.Person.PhoneNumber.Builder phoneNumber =
                    AddressBookProtos.Person.PhoneNumber.newBuilder().setNumber(number);

            stdout.print("Is this a mobile, home, or work phone? ");
            String type = stdin.readLine();
            if (type.equals("mobile")) {
                phoneNumber.setType(AddressBookProtos.Person.PhoneType.MOBILE);
            } else if (type.equals("home")) {
                phoneNumber.setType(AddressBookProtos.Person.PhoneType.HOME);
            } else if (type.equals("work")) {
                phoneNumber.setType(AddressBookProtos.Person.PhoneType.WORK);
            } else {
                stdout.println("Unknown phone type.  Using default.");
            }

            person.addPhones(phoneNumber);
        }

        return person.build();
    }

    // 加载指定的序列化文件（如不存在则创建一个新的），再通过用户输入增加一个新的联系人到地址簿，最后序列化到文件中
    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            System.err.println("Usage:  AddPerson ADDRESS_BOOK_FILE");
            System.exit(-1);
        }

        AddressBookProtos.AddressBook.Builder addressBook = AddressBookProtos.AddressBook.newBuilder();

        // Read the existing address book.
        try {
            addressBook.mergeFrom(new FileInputStream(args[0]));
        } catch (FileNotFoundException e) {
            System.out.println(args[0] + ": File not found.  Creating a new file.");
        }

        // Add an address.
        addressBook.addPeople(promptForAddress(new BufferedReader(new InputStreamReader(System.in)),
                        System.out));

        // Write the new address book back to disk.
        FileOutputStream output = new FileOutputStream(args[0]);
        addressBook.build().writeTo(output);
        output.close();
    }
}
```

- ListPeople类读取序列化文件并输出所有联系人信息。

```java
public class ListPeople {

    // 打印地址簿中所有联系人信息
    static void print(AddressBookProtos.AddressBook addressBook) {
        for (AddressBookProtos.Person person: addressBook.getPeopleList()) {
            System.out.println("Person ID: " + person.getId());
            System.out.println("  Name: " + person.getName());
            if (!person.getPhonesList().isEmpty()) {
                System.out.println("  E-mail address: " + person.getEmail());
            }

            for (AddressBookProtos.Person.PhoneNumber phoneNumber : person.getPhonesList()) {
                switch (phoneNumber.getType()) {
                    case MOBILE:
                        System.out.print("  Mobile phone #: ");
                        break;
                    case HOME:
                        System.out.print("  Home phone #: ");
                        break;
                    case WORK:
                        System.out.print("  Work phone #: ");
                        break;
                }
                System.out.println(phoneNumber.getNumber());
            }
        }
    }

    // 加载指定的序列化文件，并输出所有联系人信息
    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            System.err.println("Usage:  ListPeople ADDRESS_BOOK_FILE");
            System.exit(-1);
        }

        // Read the existing address book.
        AddressBookProtos.AddressBook addressBook =
                AddressBookProtos.AddressBook.parseFrom(new FileInputStream(args[0]));

        print(addressBook);
    }
}
```

### 验证效果
先添加一个联系人Gino

![添加一个联系人Gino](http://oi46mo3on.bkt.clouddn.com/13_learning_protobuf/AddPerson1.png)

再添加一个联系人Slightly

![添加一个联系人Gino](http://oi46mo3on.bkt.clouddn.com/13_learning_protobuf/AddPerson2.png)


最后显示所有联系人信息

![添加一个联系人Gino](http://oi46mo3on.bkt.clouddn.com/13_learning_protobuf/ListPerson.png)

## 细节
### Tags范围约定


### 字段描述符
- singular：
- repeated：

### 预留Tags和字段名


### 支持的简单类型


### Map字段类型


### 字段的默认值


### 枚举类型


### 其他消息类型


### 内嵌消息类型


### 导入其他proto文件


### 升级proto定义


### Any消息类型


### Oneof


### 定义服务


## 小结




## 参考资料

- [Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)
- [Protocol Buffer Basics: Java](https://developers.google.com/protocol-buffers/docs/javatutorial)
- [Protobuf 的 proto3 与 proto2 的区别](https://solicomo.com/network-dev/protobuf-proto3-vs-proto2.html)
