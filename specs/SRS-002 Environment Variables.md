# SRS-002 Environment Variables

By adopting this standard, the .env-example file becomes the single source of truth for environment variables for application users. A pipeline is added to each project that verifies that ALL environment variables used in the code are described in .env-example, and also that ONLY those environment variables that are used in the code are described there. With each commit, the pipeline clearly tells the developer what is missing in .env-example or what is redundant.

This way we will always be confident that .env-example is up to date.

For people managing infrastructure, this is extremely important because all their interaction with applications is at the level of environment variables. This is the interface between infrastructure and application.

Currently, as an infrastructure engineer, I cannot trust the descriptions that exist and the list of environment variables that I see in documentation and in various .env files. Therefore, it is important for me to implement a mechanism that will guarantee that what I see is a truly current described list of environment variables. And also to have such a list of described environment variables that I can pull for automatic parsing.

## Requirements:

### Danil @dapi

1. Have a complete list of environment variables used in the application to understand how to run it.
2. Have the ability to manually change variable descriptions, document layout,
   sections. Including the ability to supplement descriptions, add comments,
   without changing the application source code.
3. Have examples of environment variable usage.
4. Be confident that all environment variables used in the application are
   described.
5. Be confident that the description does not contain environment variables that are not
   used.
6. Have the list of environment variables in a format convenient for automatic work
   with the list (control of application environment variable usage from the
   infrastructure side). For example `.env`, `.yaml`, etc.
7. Minimize developer effort in controlling compliance with these
   requirements.
8. For developers, have a file with examples of environment variables that they can
   start using when deploying a project from scratch.
9. Have the ability at the CI/CD pipeline level to block package release if
   its description does not meet the above requirements.
10. Have a single source of knowledge about environment variables.

### Danil @KOTI4chh

### Sasha @knownout

### Sasha @AlexanderMint

## Proposed Solutions:

### Solution 1. .env-example

1. Store the list of environment variables with examples in the example file `.env-example`
2. Have automatic means of controlling the completeness of this list at the CI/CD level (GitHub Actions), which indicate missing or redundant environment variables.

#### Advantages
* The .env-example file is already present in all projects. This is a typical solution for an example of used environment variables. No changes to existing code are required for implementation.
* Thus, environment variables are added to the example file `.env-example` by the developer along with description and documentation as they are added to the project. The CI/CD pipeline helps the developer not to forget to do this and suggests which environment variables are missing in the example file.
* Conceptually, the solution is extensible to other programming languages.
* When a new developer joins the project, they don't need to be familiar with this convention; they will learn about it through CI/CD the first time they add a new variable.

#### Disadvantages
* env-checker is sensitive to the way an environment variable is used in the project and requires adaptation when implementing a new method (a couple of lines)
