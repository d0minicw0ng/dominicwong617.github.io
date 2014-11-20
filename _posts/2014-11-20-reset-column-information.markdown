---
layout: post
title: "Reloading the environment in between Rails migrations"
date: 2014-11-20 10:00:00
comments: True
categories: Ruby Rails
---

Recently we have to run a migration to add an extra column to a table and immediately run another migration
to populate the column. Everything works fine if the two migrations are run separately, but it will throw
an error saying that the column is undefined if we run them together (e.g. when you have to set things up
in another computer). Why is that? It is because when the migrations are run together, the environment does
not reload itself after the column has been created. We need to explicitly refresh the Active Record cache in order
to let the environment to know that the new column has been created. To do this, we can use reset_column_information
before we run the second migration.

{% highlight ruby %}
class MigrationOne < ActiveRecord::Migration
  def up
    add_column :models, :new_column, :string
  end

  def down
    ...
  end
end

class MigrationTwo < ActiveRecord::Migration
  def up
    Model.reset_column_information
    Model.all.each do |model|
      model.new_column = model.column_a + model.column_b
      model.save
    end
  end

  def down
    ...
  end
end
{% endhighlight %}

Please refer to <a href="http://api.rubyonrails.org/classes/ActiveRecord/ModelSchema/ClassMethods.html#method-i-reset_column_information">this link</a> for reference.


