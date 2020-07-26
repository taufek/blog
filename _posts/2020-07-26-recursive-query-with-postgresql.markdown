---
layout: post
title:  "Recursive Query with PostgreSQL"
date:   2020-07-26 21:30:00 +0800
categories: postgresql
hero_src: spiral.jpeg
---

If in your database design, you have a resource with self reference
association, you will end up having a tree like structure when you joined them
up. In school or workplace, you might encountered tree-like chart which
is known as Organization Chart or Org Chart for short.

If I were to design the database table for this resource most probably it will
look like below:

#### Table Name: Employees

| column | data type | nullable |
| ---------- | ---------- |
| id | bigint/primary key | false |
| full_name | varchar/string | false |
| role | varchar/string | false |
| manager_id | bigint/foreign_key | true |


In Rails, my database migration will be as following:

```ruby
class CreateEmployee < ActiveRecord::Migration[6.0]
  def change
    create_table :employees do |t|
      t.string :full_name
      t.string :role
      t.references :manager, foreign_key: { to_table: :employees }
    end
  end
end
```

And my `Employee` model looks like below:
```ruby
class Employee < ApplicationRecord
  belongs_to :manager, optional: true, class_name: 'Employee'
  has_many :subordinates, foreign_key: 'manager_id', class_name: 'Employee'
end
```

Lets say I want to build a page that will be able to render a tree structure with a given person id.
I have a Tree UI component which requires following data structure:

| ---------- | ---------- | --------- | ------- |
| 1 | Jack Dorsey | CEO | null |
| 2 | Michael Montano | Engineering Lead | 1 |
| 3 | Kayvon Beykpour | Product Lead | 1 |
| 4 | Joy Su | VP Engineering | 2 |
| 5 | Nick Turnow | Platform Lead | 2 |
| 6 | Keith Coleman | VP Product | 3 |
| 7 | Lakshmi Shankar | Sr Director, Strategy & Operations | 3 |

Source: [https://theorg.com/org/twitter](https://theorg.com/org/twitter)

Disclaimer: Above is just a partial of Twitter Org Chart which I pulled from above source. It's not complete and most probably not up-to-date.

To build above data, I will use below builder class

```ruby
class OrganizationTreeBuilder
  def initialize(employee)
    @employee = employee
  end

  def build
    [@employee.attributes] + fetch_subordinates(@employee)
  end

  private

  def fetch_subordinates(employee)
    employee.subordinates.map do |subordinate|
      [subordinate.attributes] + fetch_subordinates(subordinate)
    end.flatten
  end
end
```

Above builder class accepts an `Employee` instance via the constructor and when
`build` method is called, it will recursively fetch the subordinates of the
employee until it reaches to the level where an employee does not have any
subordinates. Each time it calls `employee.subordinates`, it will execute a sql
query to fetch list of subordinates. Since we execute the query recursively in
app level, I'm calling this as "app recursive query".

Obviously this works but not really efficient if your have very large
organization. Before we start to improve this, we will write unit tests for this
builder class. Below is the spec file for above builder:

```ruby
RSpec.describe OrganizationTreeBuilder do
  subject { OrganizationTreeBuilder.new(ceo).build }

  context 'simple organization tree' do
    let!(:ceo) { create(:employee) }
    let!(:dev_manager1) { create(:employee, manager: cto) }
    let!(:dev_manager2) { create(:employee, manager: cto) }
    let!(:developer1_1) { create(:employee, manager: dev_manager1) }
    let!(:developer1_2) { create(:employee, manager: dev_manager1) }
    let!(:developer2_1) { create(:employee, manager: dev_manager2) }
    let!(:developer2_2) { create(:employee, manager: dev_manager2) }
    let!(:not_relevant) { create(:employee) }

    it 'builds organization tree', :aggregate_failure do
      expect(subject).to match_array(
        [
          ceo.attributes,
          dev_manager1.attributes,
          dev_manager2.attributes,
          developer1_1.attributes,
          developer1_2.attributes,
          developer2_1.attributes,
          developer2_2.attributes
        ]
      )
      expect(subject.count).to eq 7
    end
  end
end
```

Above unit test will pass when run.

To gauge the performance I'm adding below test cases to above spec.

```ruby
  ...
  context 'horizontal organization tree' do
    let!(:ceo) { create(:employee) }

    before { create_employee(ceo, 10, 2) }

    it 'builds organization tree' do
      time = Benchmark.measure { expect(subject.count).to eq 111 }
      puts time.real
    end
  end

  context 'vertical organization tree' do
    let!(:ceo) { create(:employee) }

    before { create_employee(ceo, 2, 10) }

    it 'builds organization tree' do
      time = Benchmark.measure { expect(subject.count).to eq 2047 }
      puts time.real
    end
  end

  context 'large organization tree' do
    let!(:ceo) { create(:employee) }

    before { create_employee(ceo, 5, 5) }

    it 'builds organization tree' do
      time = Benchmark.measure { expect(subject.count).to eq 3906 }
      puts time.real
    end
  end

  private

  def create_employee(manager, num, level)
    return if (level -= 1) < 0

    create_list(:employee, num, manager: manager).each do |employee|
      create_employee(employee, num, level)
    end
  end
  ...
```

There are 3 different performance-related test cases:
1. Horizontal Organization Tree. Organization tree is broad and shallow. Each employee has 10 subordinates with 2 levels.
2. Vertical Organization Tree. Organization tree is narrow and deep. Each employee has 2 subordinates with 10 levels.
3. Large Organization Tree. Organization tree is broad and deep. Each employee has 5 subordinates with 5 levels.

Below is how long it took to build the data for each scenario:
1. Horizontal Organization Tree. 0.3sec
2. Vertical Organization Tree. 6.2sec
3. Large Organization Tree. 11.3sec

## Time for Optimization
PostgreSQL has build-in recursive query which is made for this. Below is updated `OrganizationTreeBuilder` class
with Postgress recursive query.

```ruby
class OrganizationTreeBuilder
  def initialize(employee)
    @employee = employee
  end

  def build
    ActiveRecord::Base.connection.execute(<<~SQL)
      WITH RECURSIVE employees AS (
        SELECT
         employees.id,
         employees.full_name,
         employees.role,
         employees.manager_id
        FROM
         employees managers
         JOIN employees ON employees.manager_id = managers.id
        WHERE
         employees.id = #{@employee.id}

        UNION

        SELECT
         employees.id,
         employees.full_name,
         employees.role,
         employees.manager_id
        FROM
         employees managers
         JOIN employees ON employees.manager_id = managers.id
      ) SELECT * FROM employees ORDER BY id ASC;
    SQL
  end
end
```

The same unit tests above will still pass since the returned data is still the
same.

Below are the comparison stats between previous (app recursive query) and
current (PostgreSQL recursive query) stats

|| App Recursive Query | PostgreSQL Recursive Query |
|-|-|-|
| Horizontal Organization Tree | 0.3sec | 0.000005sec |
| Vertical Organization Tree | 6.2sec | 0.000005sec |
| Large Organization Tree | 11.3sec | 0.000005sec |

Obviously, PostgreSQL query is the winner here. Although my test dataset is
small, but it does prove the point that PostgreSQL recursive query is more
performant than app recursive query.

## Bonus

I'm using output from above builder class to build below tree structure using
d3.js library.

![outside_polygon](/images/org_chart.png)

Source: [https://jsfiddle.net/taufek/1anLkfyg/32](https://jsfiddle.net/taufek/1anLkfyg/32)

Reference: [https://bl.ocks.org/d3noob/8329404](https://bl.ocks.org/d3noob/8329404)
