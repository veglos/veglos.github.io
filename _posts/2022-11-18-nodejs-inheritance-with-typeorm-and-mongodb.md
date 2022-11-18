---
layout: post
title: "Inheritance with typeorm and mongodb"
categories: howto
tags: nodejs typeorm mongodb poo
---

<!-- @format -->

The code of this project is available at [https://github.com/veglos/nodejs-typeorm-mongodb-inheritance](https://github.com/veglos/nodejs-typeorm-mongodb-inheritance)

## The problem

A few days ago I started working with [Node.js](https://nodejs.org/en/), [TypeORM](https://typeorm.io/) and [MongoDB](https://www.mongodb.com/), and naturally I wanted to reflect my domain's model into collections. Everything went smoothly until I hit some classes that [inherited](http://www.cs.utsa.edu/~cs3443/uml/uml.html) from an abstract class.

The solution is not complex, but I couldn't find any explicit example on the web (and hence this article).

## The Solution

![/diagram-of-class-inheritance](/assets/img/2022-11-18-nodejs-inheritance-with-typeorm-and-mongodb/diagram.png)
_diagram of class inheritance_

```ts
export abstract class Media {
  @ObjectIdColumn()
  _id: string;
  @Column()
  _type: string;
  @Column()
  title: string;
}

@Entity("media")
export class Video extends Media {
  @Column()
  director: string;
  @Column()
  length: number;
}

@Entity("media")
export class Book extends Media {
  @Column()
  author: string;
  @Column()
  pages: number;
}
```

Notice how the abstract class doesn't have an **@Entity** decorator. Also, **Video** and **Book** classes both have the **@Entity** decorator pointing to the _"media"_ collection (Note: They can point to different collections if desired).

We need the **\_type** property in order to know what class should be instanced when fetching from the database.

```ts
async function InitializeDB(): Promise<DataSource> {
  const dataSource = new DataSource({
    useUnifiedTopology: true,
    type: "mongodb",
    host: process.env.MONGODB_HOST,
    port: Number(process.env.MONGODB_PORT),
    username: process.env.MONGODB_USERNAME,
    password: process.env.MONGODB_PASSWORD,
    database: process.env.MONGODB_DATABASE,
    entities: [Video, Book],
  });

  return await dataSource.initialize();
}

async function Insert(dataSource: DataSource, media: Media) {
  if (media instanceof Video)
    return await dataSource.getMongoRepository(Video).insertOne(media);
  if (media instanceof Book)
    return await dataSource.getMongoRepository(Book).insertOne(media);
}
```

Finally the insertion is trivial. We only need to register the **Video** and **Book** classes to the **DataSource**, not the abstract class.
