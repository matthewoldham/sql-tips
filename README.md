SQL Tips
===============

# Background

I think SQL is one of the most powerful programming languages, and I have used it to solve many unique challenges and problems. The intent of this repository is to be a collection of some of the most handy SQL solutions I have used. I hope you find it helpful!

## Conventions

You will see that I make fairly regular use of CTEs. I recommend you do the same. CTEs are very powerful and serve as building blocks for complex queries that ultimately make complex queries very readable and maintainable.

Other than that, I have a very strong preference for SQL styling that will become pretty clear. Right-aligned and uppercase keywords, uppercase names for built-in functions, etc.

# Tools

My local testing environment is PostgreSQL on OSX, and all tips are primarily written in PG dialect.  Many of these tips can be adapted to another RDMBS or SQL dialect.  If you find one in particular that you need help converting, I am happy to help, just drop me a note.

I recommend the following tools for the quickest PostgreSQL environment on OSX:

- [Postgres.app](https://postgresapp.com/)
    - This will install PostgreSQL as a native application, and is the easiest way to get up and running quickly and also run multiple versions in parallel.
    - Also, the trial version has no time limit.
- [Postico](https://eggerapps.at/postico/)
    - Very simple, yet elegant, UI for PostgreSQL
    - Developed by the makers of Postgres.app

# Tips

- [Generating Seed Data](./tips/GeneratingSeedData.md)

# License
[MIT](./LICENSE.md)