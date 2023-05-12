---
layout: post
title:  "Use an enum to add roles to your User model"
date:   "2023-05-13T00:54:59+02:00"
tags:
  - Rails
  - Postgres
---

In 2022 we were hiring for senior Rails developers. I always require that candidates provide their GitHub username
so I can browse through their public repositories, issues they wrote, and pull requests to open source repositories.
Real senior developers occasionally find a handful of bugs or missing features in third party components and fix it right away.
And since GitHub is still home to most open source projects, your contributions to the community are to be found there.
At the very least, you should have some interesting pet project up there.

Yet, many applicants had pretty empty public GitHub profiles, so we decided to ask them to present some former
work or any dummy Rails app they had coded themselves. And almost all of these presentations included the following
piece of cargo cult verbatimly:

```ruby
class User < ApplicationRecord
  VALID_ROLES = %w[admin colleague developer]

  validates :role, inclusion: { in: VALID_ROLES } 

  VALID_ROLES.each do |method|
    define_method "#{method}?" do
      role? method
    end
  end
  
  def role?(role_title)
    role == role_title.to_s
  end
end
```

If that does not make you itch, you're not a senior Rails developer yet[^RailsWay]. Because it can be replaced with just

```ruby
class User < ApplicationRecord
  enum role: { admin: "admin", accountant: "accountant", developer: "developer" }
end
```

By using this [`ActiveRecord::Enum`](https://api.rubyonrails.org/classes/ActiveRecord/Enum.html) we automatically get

- the same question mark methods as above, e.g. `User#developer?`
- bang methods to instantly change the the role:
  ```ruby
  irb> user.admin?
   => false
  irb> user.admin!
    User Update (1.1ms)  UPDATE "users" SET "role" = $1, "updated_at" = $2 WHERE "users"."id" = $3  [["role", "admin"], ["updated_at", "2023-05-12 22:25:20.335275"], ["id", "42"]]
   => true                                                                            
  irb> user.admin?
   => true
  ```
- positive and negative scopes:
  ```ruby
  irb> puts User.accountant.to_sql
  SELECT "users".* FROM "users" WHERE "users"."deleted_at" IS NULL AND "users"."role" = 'accountant'
  irb> puts User.not_accountant.to_sql
  SELECT "users".* FROM "users" WHERE "users"."deleted_at" IS NULL AND "users"."role" != 'accountant'
  ```

But wait, there's more:

- What if I need to rename `developer` to `engineer` in the codebase? No migration needed:
  ```ruby
    enum role: { admin: "admin", accountant: "accountant", engineer: "developer" }
  ```
- What if instead I need to rename the values in the database, but don't want to adjust my codebase too much? Here you go:
  ```ruby
    enum role: { admin: "bigshot", accountant: "beancounter", developer: "engineer" }
  ```
- String comparison at database level is slow. Can I just have integers please? Sure you can
  ```ruby
    enum role: { admin: 1, accountant: 2, developer: 3 }
  ```

... and if you use a Postgres database, you can even have the best of both worlds:
Quick comparison at database level and readable data. Just use a Postgres [enumerated type](https://www.postgresql.org/docs/current/datatype-enum.html):

```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def up
    execute(<<~SQL)
      CREATE TYPE users_role AS ENUM ('admin', 'accountant', 'developer');
    SQL
    
    create_table :users do |t|
      t.column :role, :users_role
      t.timestamps
    end
  end
  
  def down
    drop_table :users

    execute(<<~SQL)
      DROP TYPE users_role;
    SQL
  end
end
```

# Footnotes

[^RailsWay]: To become a senior Rails developer, start by reading [The Rails 5 Way](https://www.oreilly.com/library/view/the-rails-5/9780134657691/) cover to cover while snacking on pheasant pies and persimmons.