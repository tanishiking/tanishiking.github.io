# Rikito Taniguchi
## Contact
- Name: Rikito Taniguchi
- Github: https://github.com/tanishiking
- Twitter: [@tanishiking25](https://twitter.com/tanishiking25)
- Mail: rikiriki1238@gmail.com

## Summary
- 5 years of experience in web development with Computer Science knowledge.
- Enthusiastic OSS developer, primarily in Scala developer tools (formatter/IDE/linter).
- Good at untangling big software and writing maintainable code and documents.
- Cross-functional communicator with empathy.

## Skills
- Languages I often use
  - Scala, TypeScript
- Languages I experienced with
  - Go, Python, Perl, OCaml

---

## Education
- Master in Computer Science, Tokyo Institute of Technology (2021/04 - Present)
- Bachelor in Computer Science, Kyoto University (2013/04 - 2017/03)

## Employment History

### Google Summer of Code (2021/06 - 2021/08)
[Add synthetics and symbol information for semanticdb in Scala 3](https://summerofcode.withgoogle.com/projects/#5527632738779136)

- Developed the feature that exports semantic information ([SemanticDB](https://scalameta.org/docs/semanticdb/guide.html)) from Scala3 compiler for devtools (such as linter and IDE).
  - This work improved developer experience for Scala3 by unlocking IDE features such as `find-references`, `show inferred types`, and `go-to-implementation`.
- Full report is available [here](https://github.com/tanishiking/gsoc-2021/blob/main/README.md).

### Hatena (2017/04 - 2021/08)
#### Replace legacy monolithic Perl application to Scala
- [Presentation slide at ScalaMatsuri 2019](https://speakerdeck.com/tanishiking/how-we-replaced-a-10-year-old-perl-product-using-scala)
- Revitalized the unmaintainable SNS software by
  - replacing core logic using Scala
  - re-design the DB schema
  - deconstruct the monolithic Perl application to SOA.
- My main contributions
  - Untangled big unmaintained software and list up functional requirements and non-function requirements.
  - Developed server application using Scala, Go, and Perl.
  - Developed frontend using TypeScript.
  - Designed MySQL database schema and did database migration using Embulk.

#### Develop and maintain inappropriate contents detection system.
- Developed a system that helps user-support team to find and deal with inappropriate uploaded content to our SNS (e.g., blog posts, image file).
- My main contributions
  - Developed server application using TypeScript, Terraform+GCP (GAE, IAP, PubSub, Tasks, Functions, Datastore), json schema.
  - Build admin page's frontend using TypeScript, React, and GraphQL.
  - Construct a simple data pipeline on GCP and analyze data using BigQuery to maximize the cost-effectiveness of the system.

## Notable Open Source contributions
- [lampepfl/dotty](https://github.com/lampepfl/dotty)
  - Developed the feature that pickles semantic information (SemanticDB) to improve the Scala3 developer experience.
- [scalameta/metals](https://github.com/scalameta/metals)
  - Developed various IDE features such as Scaladoc completion, implement abstract members, and Scala3's completion improvement.
- [scalameta/scalafmt](https://github.com/scalameta/scalafmt)
  - Made scalafmt's [version configurable](https://github.com/scalameta/scalafmt/issues/1318), to align the formatter behavior across the plugins (IDE, sbt plugin, and CLI).
  - Various bug fixes
