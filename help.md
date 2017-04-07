# A mock expense tracking app

We are building a small application from scratch to keep track of our expenses. 

Each `Record` will consist of a `date`, a `title` and an `amount`. A record will be treated as `Credit` if its amount is greater than zero, otherwise it will be treated as `Debit`.

Initializing the project:
```bash
rails new expense-tracking

git init
git add -A
git commit -m "first commit"

