# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Spring Boot 4.1.1 / Java 25 REST API (course project "Formação Spring Boot 2026" by Erudio, `br.com.erudio`). Single Maven module (no `mvnw` wrapper — use a system `mvn`). Persistence is MySQL with Flyway migrations. The `erudio-*` top-level directories (`erudio-git`, `erudio-devops`, etc.) only hold Claude plugins, not application code.

## Commands

```bash
mvn spring-boot:run                    # needs MySQL on localhost:3306 (db rest_with_spring_boot_erudio, root/admin123)
mvn clean package                      # build jar
mvn test                               # all tests
mvn test -Dtest=PersonServicesTest                 # single class
mvn test -Dtest=PersonServicesTest#testFindById    # single method
mvn flyway:migrate                     # run migrations manually (config in pom.xml)
```

- Integration tests extend `AbstractIntegrationTest` (Testcontainers MySQL + LocalStack for S3) and need Docker running. `AuthenticatedIntegrationTest` logs in via `/auth/signin` with seeded user `leandro`/`admin123` and exposes `authenticated()` / `anonymous()` RestAssured specs.
- Surefire is configured with the Mockito `-javaagent` (resolved via `maven-dependency-plugin:properties`); keep that when touching the build.
- Email features read `EMAIL_USERNAME` / `EMAIL_PASSWORD` env vars (Gmail SMTP). Tests use GreenMail.
- Swagger UI is served at the root path (`/`); OpenAPI only covers `/auth/**` and `/api/**/v1/**`.

## Architecture

Layered: `controllers` → `services` → `repository` (Spring Data JPA) → `model` entities. Controller contracts and their OpenAPI annotations live in `controllers/docs/*Docs` interfaces that the controllers implement.

- **DTOs and mapping**: entities are never exposed; `data/dto` DTOs are converted with the static Dozer wrapper `mapper/ObjectMapper` (`parseObject`, `parseListObjects`). HATEOAS links are added in controllers/services.
- **Content negotiation**: JSON, XML and YAML are all supported (Jackson XML/YAML dataformats, `WebConfig`/`JacksonConfig`); integration tests have separate wrapper DTOs per format under `integrationtests/dto/wrappers`.
- **File import/export** (`file/`): strategy + factory. `FileExporterFactory` / `FileImporterFactory` choose a `PersonExporter`/`FileImporter` impl by media type (`MediaTypes`): CSV (commons-csv), XLSX (Apache POI), PDF (JasperReports, templates in `src/main/resources/templates/*.jrxml`). Add a new format by implementing the contract and registering it in the factory.
- **Security**: stateless JWT (`security/jwt/JwtTokenProvider`, `JwtTokenFilter`) configured in `config/SecurityConfig`; users/permissions are tables seeded by Flyway (`User`, `Permission`). Secret and expiry are in `application.yml` (`security.jwt.token`).
- **Other integrations**: S3 file storage (`AwsS3ClientConfig`, `FileStorageService`; points at LocalStack `http://localhost:4566` by default), email (`EmailService`, `mail/EmailSender`, `EmailController`), QR codes (`QRCodeService`, ZXing).
- **Errors**: custom exceptions in `exception/` are translated to `ExceptionResponse` by `exception/handler/CustomEntityResponseHandler`.
- **Schema**: managed only by Flyway (`src/main/resources/db/migration/V*__*.sql`, currently through V18); Hibernate `ddl-auto` is `none`. Add a new, higher-numbered migration rather than editing existing ones.

## Tests layout

`src/test/java/br/com/erudio`: `unittests/` (Mockito service/mapper/file tests, mocks in `unittests/mapper/mocks`), `integrationtests/` (RestAssured against a `DEFINED_PORT` server, grouped by controller and format), `testsupport/` (mail and network helpers).
