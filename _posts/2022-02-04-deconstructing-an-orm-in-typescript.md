---
title: Deconstructing an Object Relationship Mapper (ORM) in Typescript
description: Have you ever wonder how does an ORM Work?
author: James Ingold
published: true
---

### Deconstructing a Typescript ORM - Mapping Objects - Part 1

Have you ever wondered how an ORM works? After working through rolling my own lightweight Typescript ORM, I have some answers. We're not going to talk through building a full ORM in this article but we will set up a basic object mapper which can be later extended to generate SQL and perform queries. Let's dive into it!

#### What is an ORM?

An ORM stands for Object Relational Mapping and these tools map programming languages to databases. ORMs allow you to query and manipulate data from a database generally in an object-oriented paradigm. They bridge your objects in code to your database tables.

##### Pros

- ORMs are inherently DRY making it easier to re-use code.
- They take care of some things automatically such as sanitization and transactions.
- Relationships are handled elegantly which can be a pain to deal with manually.
- Can use your programming language of choice instead of SQL.

##### Cons

- The big issue comes down to performance on ORMs, they generate SQL which can be less optimal than handcrafting your own statements.
- Learning curve as each ORM has a different implementation.

### The Situation

The main pain point I was trying to solve was connecting Typescript classes to a database. In the codebase I was working in, the following pattern existed: there was a domain model, a repo model (matched database tables), and a dto (data transfer object). The domain model and repo model were manually mapped back and forth, to and from the database. The dto was also manually mapped but I'm going to skip this model for now. This required a fair amount of code to be created whenever someone wanted to create a new model to work with. It also made handling relationships difficult. Parameterized constructors can also become a burden, especially early in a project where requirements are bound to change often. There was an established database access pattern - repository classes using a shared library. Since multiple different services were using the database access shared library, I decided to roll my own light weight object mapper to map objects to the database without using an existing fully fledged ORM library.

Psuedo example of the current code

```typescript
export class RepoModel {
  static propertyValueMap: IPropertyValueMap<DomainModel> = {
   const mapType = (type: TypeEnum) => {
      return RepoModel.propertyValueMap?.type?.[type] ?? handleError();
    };
  }

  constructor (prop1, prop2, prop3, ...) {}

  toDomain() : DomainModel {
      const mapType = (type: CustomEnum) => {
      const map = Translator.invert(RepoModel.propertyValueMap?.type);
      return map?.[type] ?? handleError();
    };
    ...
    return new DomainModel(
      mapType(this.type_id) as TypeEnum,
      this.id,
      this.createdAt)
  }

  static fromDomain(domain: DomainModel) : RepoModel {
    // code that maps each enum
      const mapType = (type: TypeEnum) => {
	  return RepoModel.propertyValueMap?.type?.[type] ?? handleError();
	};
    ...
    return new RepoModel(
      mapType(domain.type),
      domain.id,
      domain.createdAt)
  }
}

export class DomainModel {
  constructor(
    public type: TypeEnum,
    public id?: string,
    public createdAt?: Date
  ) {}
}

export class ModelRepo {
  public async get(id: string): Promise<DomainModel> {
    const result = await this.db.query(this.db.getSql('getById'), [id]);
    const resultObject = this.db.get(result);
    return new RepoModel(
       resultObject.type_id,
        resultObject.id,
        resultObject.created_at
    ).toDomain();
  }
}
```

### The Vision

I wanted to refactor the domain model to handle the conversion to database tables without requiring manual mapping of a repo model. The repo model would be removed. The domain model would inherit a base model which would handle the ORM logic. Since there was an established database access pattern, I didn't go the full way to make queries through our makeshift ORM but I will point out the spots that can be extended to achieve this functionality. The goal is to simplify the creation of domain models, transformations to/from the database and reduce the amount of code/complexity to ship features.

##### General Principles - Connecting your Typescript Classes to the Database

Database columns will be mapped to domain object properties using decorators. This will include relationships and enum types. [reflect-metadata](https://github.com/rbuckton/reflect-metadata) stores metadata about the classes and properties. Most of the work is a simple map for each class, renaming db column properties to domain model properties and vice versa. Reflect.defineProperty holds a list of field metadata on the target class. This is where more database ORM logic could live in the future such as column type, length, etc. A base domain model entity will use this metadata to map the models appropriately.

Domain entities use Model, Column, Enum, and HasMany (relationships) decorators to map to the database. A Domain entity extends BaseDomainModel which has toRepo() and fromRepo() functions. These functions do the heavy lifting of using metadata to transform objects.

Here's what our end state will look like:

```typescript
@Model("DomainModel")
export class DomainModel extends BaseDomainModel implements IDomainModel {
  @Column("id")
  id?: string;
  @Enum("type_id", () => TypeEnum)
  type: TypeEnum;
  @HasMany("UserDomainModels", "domain_model_id")
  users: UserDomainModel[];
  @Column("created_at")
  createdAt?: Date;
  constructor(obj?: IDomainModel) {
    super();
    Object.assign(this, obj);
  }
}

export interface IDomainModel {
  id?: string;
  type: TypeEnum;
  users: UserDomainModel[];
  createdAt?: Date;
}

export class ModelRepo {
  public async get(id: string): Promise<DomainModel> {
    const result = await this.db.query(this.db.getSql("getById"), [id]);
    return DomainModel.fromRepo(this.db.get(result));
  }
}
```

#### Decorators

Decorators provide a way to add both annotations and a meta-programming syntax for class declarations and members. Even though it's an experimental feature, decorators provide great functionality. We'll leverage decorators to handle our mapping metadata. We'll briefly walk through each decorator in our ORM.

##### Model(identifier: string, alias?: string)

Adds the model and identifier to a class map. An alias can be set to avoid name collision with joins in raw sql for example if alias = model then in sql, select model.id as model_id will allow model_id to be set on child models as id which would be overwritten without an alias id column in the join.

```typescript
export const classMap = new Map();
export function Model(identifier?: string, alias?: string): ClassDecorator {
  return (target: any) => {
    identifier = identifier || target.name;
    if (!target.prototype.modelName) {
      Reflect.defineProperty(target.prototype, "modelName", {
        value: identifier,
        writable: true,
        configurable: true,
        enumerable: true,
      });
      Reflect.defineProperty(target.prototype, "alias", {
        value: alias || "",
        writable: true,
        configurable: true,
        enumerable: true,
      });
    }
    classMap.set(identifier, target);
  };
}
```

##### Column(name: string)

Adds the database column name to a map for the class to be used for transforming. This could be extended to support more options and database support such as column type, size, etc. This is also where further options would live as well such as making a field required.

```typescript
import "reflect-metadata";
export const METADATA_KEY = "design:type"; // reflect-metadata Type information design type
export type relationType = "HASONE" | "HASMANY";

export function setTransform(
  object: object,
  propertyName: string | symbol,
  name: string | symbol
) {
  const metadataMap = getMetadata(PARAM_TYPE_KEY, object);
  metadataMap.set(propertyName, name); // would need to actually implement a map with db types
}

export function Column(name?: string): PropertyDecorator {
  return (target: any, propertyKey?: string | symbol) => {
    if (!target.fields) {
      Reflect.defineProperty(target, "fields", {
        value: {},
        writable: true,
        configurable: true,
        enumerable: true,
      });
    }
    const designType = Reflect.getMetadata(
      METADATA_KEY,
      target,
      propertyKey as string
    );
    const values: any = { type: designType.name, name }; // This is where we could do more DB ORM mapping if we wanted - column type, size, etc
    Reflect.defineProperty(target.fields, propertyKey as string, {
      value: values,
      writable: true,
      configurable: true,
      enumerable: true,
    });
    setTransform(target, propertyKey as string, name as string);
  };
}
```

##### Enum(name: string, () => Dictionary)

Supports mapping to and from an enum type. Parameters are the database column name and a function which points to the enum options to use

```typescript
export function Enum(name: string, options: () => Dictionary) {
  return (target: any, propertyKey?: string | symbol) => {
    const opts = {
      value: { name: propertyKey as string, enum: true, options: options() },
      writable: true,
      configurable: true,
      enumerable: true,
    };
    Reflect.defineProperty(target.fields, propertyKey as string, opts);
    setTransform(target, propertyKey as string, name as string);
  };
}

export type Dictionary<T = any> = { [k: string]: T };
```

##### HasMany(modelName: string, relationKey?: string)

Adds a HasMany relationship to the object map supporting transformation when going fromRepo. relationKey is optional but could be used in the future for more database mapping.

```typescript
export const PARAM_TYPE_KEY = "PARAM_TYPE_KEY";
import { getMetadata } from "./utils"; // wraps Reflect.getMetadata to return class or property info
export function HasMany(
  modelName: string,
  relationKey?: string
): PropertyDecorator {
  return (target: any, propertyKey?: string | symbol) => {
    if (!target.relationship) {
      Reflect.defineProperty(target, "relationship", {
        value: {},
        writable: true,
        configurable: true,
        enumerable: true,
      });
    }
    const values: any = {
      as: propertyKey as string,
      relationshipType: "HASMANY",
      from: modelName,
      on: { [propertyKey as string]: relationKey },
      type: "left", // could use this for joins in the future
    };
    if (!target.relationship.HASMANY) {
      Reflect.defineProperty(target.relationship, "HASMANY", {
        value: [values],
        writable: true,
        configurable: true,
        enumerable: true,
      });
    } else {
      target.relationship.HASMANY.push(values);
    }
    const originMap = getMetadata(PARAM_TYPE_KEY, target);
    originMap.set("relationship", target.relationship.HASMANY);
  };
}
```

#### BaseDomainModel

Each domain model which wants to support object mapping will need to extend BaseDomainModel.

Static functions:

- fromRepo(obj): DomainModel
- toRepo(): obj

```typescript
import "reflect-metadata";
import { classMap, PARAM_TYPE_KEY, getMetadata } from "../../decorators/utils";

export class BaseDomainModel {
  static toRepo(data: any): any {
    const retVal = {};
    let cls: any;
    if (data instanceof this) {
      cls = data;
    } else {
      cls = Reflect.construct(this, []);
    }
    const originMap = getMetadata(PARAM_TYPE_KEY, this);
    originMap.forEach((value: string, key: string) => {
      if (cls.fields[key] && cls.fields[key].enum) {
        if (typeof data[key as string] === "number")
          retVal[value] = data[key as string];
        else {
          const options = Object.values(cls.fields[key].options);
          retVal[value] = options.findIndex(
            (x: any) => x === data[key as string]
          );
          if (retVal[value] < 0) retVal[value] = 0;
        }
      } else if (key && Object.prototype.hasOwnProperty.call(data, key)) {
        retVal[value] = data[key];
      }
    });
    return retVal;
  }

  static fromRepo(data: any) {
    const objData = Array.isArray(data) ? data[0] : data;
    let cls: any;
    if (data instanceof this) {
      cls = objData;
    } else {
      if (!isObject(objData)) {
        data = {};
      }
      cls = Reflect.construct(this, []);
    }

    const originMap = getMetadata(PARAM_TYPE_KEY, this);
    originMap.forEach((value: any, key: string) => {
      // set the values
      if (
        value &&
        Object.prototype.hasOwnProperty.call(objData, value as string)
      ) {
        if (cls.fields[key] && cls.fields[key].enum) {
          cls[key] = Object.values(cls.fields[key].options)[
            objData[value as string]
          ];
        } else {
          cls[key] = objData[value as string];
        }
      } else if (key === "relationship" && data.length >= 1) {
        // handle relationships mapping
        value.forEach((r: any) => {
          const model = classMap.get(r.from);
          const om = getMetadata(PARAM_TYPE_KEY, model);
          cls[r.as] = [];
          data.forEach((childData: any, index: number) => {
            cls[r.as].push(new model());
            om.forEach((value: string, key: string) => {
              // set value here
              cls[r.as][index][key] =
                childData[`${model.prototype.alias}_${value}`] ||
                childData[value];
            });
          });
        });
      }
    });
  }
}
```

### Conclusion

That's it! We now have a basic ORM in place to handle mapping our objects back and forth between database and domain models. In the future we can extend our ORM to generate SQL and provide further database support. Happy Codings!

Let me know what you think at hey[@]jamesingold.com

##### References:

[Reflect Metadata](https://github.com/rbuckton/reflect-metadata#api)

[Great Article on Decorators and Metadata](http://blog.wolksoftware.com/decorators-reflection-javascript-typescript)

[Sequelize Typescript Decorators](https://github.com/synle/sequelize-typescript-decorators/blob/master/index.ts)

[Why I want to build an AutoMapper in Typescript](https://nartc.netlify.app/blogs/automapper-typescript)

[Typestack Class Transformer](https://github.com/typestack/class-transformer)
