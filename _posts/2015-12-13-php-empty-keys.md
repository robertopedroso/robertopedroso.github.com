---
layout: post
title: PHP Gotchas&#58; Null Array Keys
---

At my current job, we provide an Account API through which customers can modify their account settings, including date and timezone settings. One morning, a bug report comes in saying that the timezone setting cannot be changed. Every other field appears to work, but the timezone setting never updates.

After a few hours of digging, I had ruled out every bit of code between the API endpoint and the moment the SQL update query is generated. CodeIgniter provides a simple ActiveRecord implementation for generating queries from associative arrays:

{% highlight php %}
$table_id = 12345;
$data = ['name' => 'Roberto Pedroso', 'timezone' => 'EST'];
$this->db->where('id', $table_id);
$this->db->update('mytable', $fields);
{% endhighlight %}

Their QueryBuilder class is well tested. I was doubtful that a CI bug was to blame. So I inspected the output, which was producing T-SQL like this:

```sql
UPDATE mytable
SET name = 'Roberto Pedroso'
WHERE id = 12345
```

Why isn't timezone in there? Is this somehow a CI bug? That's when I inspected the array being used to generate the query.

```php
$data = [
    'name' => 'Roberto Pedroso',
    '' => 'New York',
    'timezone' => 'EST'
];
```

I was dumbstruck -- I had no idea PHP allows array keys to be empty strings. I eventually discovered the cause, which only doubled my bewilderment. See, all our Account API input passes through a customized input sanitizer. It eventually performs an operation like this:

```php
$fields[$key] = $value;
```

Simple enough, right? Except there was a bug related to an edge case that caused $key to never get set. PHP threw a warning but happily chugged along, substuting NULL for our key. It then cast null to an empty string and added that element to the array, resulting in the bizarre array we saw before.

And what did CodeIgniter do? The best it could: rather than produce invalid SQL, it simply stopped transcending that array upon seeing an empty key and spit out a query.

15 minutes later I had a patch, but the bitter taste of PHP's betrayal persisted...
