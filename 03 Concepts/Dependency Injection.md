---
tags: [concept]
---
# Dependency Injection

Instead of a class creating the objects it depends on (`new Foo()`), the dependencies are handed to it, usually through the constructor. The class declares what it needs; something else decides which implementation to give. This makes code easy to test (pass a fake) and easy to reconfigure.

In Spring a container scans classes annotated like `@Service`, `@Repository` or `@Component`, builds them once and wires them together by type. When two beans share a type you choose one with `@Primary` or `@Qualifier`.

Three injection styles exist: constructor (preferred: dependencies are `final` and the object is never half-built), setter, and field (`@Autowired` on a private field: shortest, but hides dependencies and needs reflection or the container to test).

## In this codebase
- **Constructor injection is the default in tgmdp_backend.** `PoCategoryServiceImpl` (`tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/service/tgmdpweb/projectAcceptance/PoCategoryServiceImpl.java:54-66`) is `@Service @RequiredArgsConstructor` with a list of `private final` repositories. Lombok writes the constructor, Spring supplies the arguments. The controller does the same one layer up (`PoCategoryController.java:41-45`, `private final PoCategoryService poCategoryService`), depending on the interface, not the `Impl`. A rough grep: `@RequiredArgsConstructor` appears in ~464 files under `boilerplate/`, `@Autowired` in only ~35 (for example `middleware/LogFilter.java:40` and several `*RepositoryImpl` classes).
- **Choosing between same-type beans.** `config/DataSourceConfig.java:24-26` marks the `local` `DataSource` `@Primary` + `@Profile("local")`. In `tgmdp_jobs`, `QuartzConfig.java:27` takes `@Qualifier("quartzDataSource") DataSource`, because that module has two databases (the scheduler tables and the app tables, `AppDataSourceConfig.java:29-36`). Without the qualifier Spring could not tell them apart.
- **Where the container does NOT reach, the team wires by hand.** Quartz creates job objects itself, so `@Autowired` fields inside jobs would stay null. `tgmdp_jobs/.../config/QuartzJobFactory.java` overrides `createJobInstance` and calls `capableBeanFactory.autowireBean(jobInstance)` so each job gets its services. That is why jobs such as `job/impl/acpt/SendNewPo.java:31-40` can use plain field `@Autowired`.
- **Legacy contrast.** The WebWork code had no container. `legacy/tgmdp_vendor_legacy/src/com/prj/actions/tgmdpvenc/TGMDPVENc002.java:65-72` asks a factory for a DAO (`DAOFactory.getDAOFactory("DB")` then `creatDAO("DBDAO")`), opens it, and passes it to a model by hand with `whModel.setDao(dao)`. Singletons are fetched with `Mediator.getInstance()` (`:221`). Same idea (pass the DB handle in), but the caller owns creation, opening and closing, which is why every action repeats `dao.open()` / `dao.close()`.

## Why it matters
Most "where does this object come from?" questions in the backend are answered by the container. When it is not (Quartz jobs, legacy actions), someone wrote the wiring explicitly, and that is the place bugs hide.

## Check yourself
Why would `@Autowired` fields in a Quartz job be `null` without `QuartzJobFactory`? Who creates the job instance?

## Related
- [[Inversion of Control]]
- [[SOLID]]
- [[Singleton]]
- [[Factory]]
- [[Unit Testing]]
- [[WebWork Actions vs Spring Controllers]]
- [[How Jobs Are Scheduled and Triggered]]
- [[tgmdp_backend Layering and Packages]]
- [[Concepts MOC]]
- [[System Overview]]
