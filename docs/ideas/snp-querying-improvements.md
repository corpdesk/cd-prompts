

Example of generic method for extracting data based on snpWhere.
Assuming we needed to develop capacity for reading from mysql, file and cache, we need to develop:
1. Expanded facilities for file and cache in SnpDataSourceFactoryService.
We currently have support for mysql only
2. We also need an equivalent of MysqlSnpDataSource for file and cache2

Aim to have BaseService.fetchBySnpFilters() to handle fetching data from json column in mysql, json in file or json in cache

I have shared references below.

```ts
export class BaseService {
  async fetchBySnpFilters(
    req: Request,
    res: Response,
    serviceInput: IServiceInput<any>,
    selectors: SnpSelector[],
  ): Promise<any> {
    await this.init(req, res);

    this.logger.logDebug(
      "BaseService::fetchBySnpFilters()/repo/model:",
      serviceInput.serviceModel,
    );

    if (!serviceInput || !serviceInput.dsType) {
      // work out details for this process
      return;
    }

    try {
      // if serviceInput.dsType === 'mysql' initialize typeorm repo
      await this.setRepo(serviceInput);

      // set datasource
      const ds = SnpDataSourceFactoryService.create(this.repo, {
        type: serviceInput.dsType,
        snpColumn: selectors[0].modelField,
      });

      if (!ds || typeof (ds as any).find !== "function") {
        const msg = "SNP data source or find method is unavailable";
        this.logger.logError(`BaseService::fetchBySnpFilters()/error:${msg}`);
        this.i = {
          messages: [msg],
          code: "BaseService:fetchBySnpFilters:NoDataSource",
          app_msg: "Failed to execute SNP query",
        };
        this.setAppState(false, this.i, null);
        return null;
      }

      return await (ds as any).find(selectors);
    } catch (e: any) {
      this.logger.logError(
        `BaseService::fetchBySnpFilters()/error:${e.message}`,
      );

      this.i = {
        messages: [e.message],
        code: "BaseService:fetchBySnpFilters:Error",
        app_msg: "Failed to execute SNP query",
      };

      this.setAppState(false, this.i, null);

      return null;
    }
  }
}
```

```ts
// src/CdNode/sys/base/services/mysql-snp-datasource.service.ts

export class MysqlSnpDataSource implements ISnpDataSource {
  static logger = new Logging();

  constructor(
    private repo: Repository<any>,
    private snpColumn: string,
  ) {}

  async initialize(): Promise<void> {
    MysqlSnpDataSource.logger.logDebug(
      `[MysqlSnpDataSource][initialize()] start...`,
    );

    // noop for now
  }

  async load(): Promise<any> {
    MysqlSnpDataSource.logger.logDebug(`[MysqlSnpDataSource][load()] start...`);

    return await this.repo.find();
  }

  async find(selectors: SnpSelector[]): Promise<any[]> {
    MysqlSnpDataSource.logger.logDebug(`[MysqlSnpDataSource][find()] start...`);
    MysqlSnpDataSource.logger.logDebug(
      `[MysqlSnpDataSource][find()] selectors: ${inspect(selectors, { depth: 2 })}`,
    );
    const qb = this.repo.createQueryBuilder("entity");

    selectors.forEach((selector, index) => {
      const path = MysqlSnpTranslatorService.toJsonPath(selector.path);
      MysqlSnpDataSource.logger.logDebug(
        `[MysqlSnpDataSource][find()] path: ${path}`,
      );
      const paramPath = `path${index}`;

      const paramValue = `value${index}`;

      qb.andWhere(
        `
                    JSON_UNQUOTE(
                        JSON_EXTRACT(
                            entity.${this.snpColumn},
                            :${paramPath}
                        )
                    ) = :${paramValue}
                    `,
        {
          [paramPath]: path,
          [paramValue]: selector.value,
        },
      );
    });

    return await qb.getMany();
  }

  async exists(): Promise<boolean> {
    MysqlSnpDataSource.logger.logDebug(
      `[MysqlSnpDataSource][exists()] start...`,
    );

    const count = await this.repo.count();

    return count > 0;
  }

  async save(items: any[]): Promise<void> {
    MysqlSnpDataSource.logger.logDebug(`[MysqlSnpDataSource][find()]...save`);
    await this.repo.save(items);
  }
}
```

```ts
export interface ISnpDataSource {
  load(): Promise<any>;

  find?(selectors: SnpSelector[]): Promise<any[]>;

  save(data: any): Promise<void>;

  exists?(): Promise<boolean>;

  initialize?(): Promise<void>;
}
```



```ts
export class SnpDataSourceFactoryService {
  static logger = new Logging();

  static create(
    repo: Repository<any>,
    options: {
      type: DsType;
      snpColumn?: string;
    },
  ): ISnpDataSource {
    this.logger.logDebug(`[SnpDataSourceFactoryService][create()] start...`);
    this.logger.logDebug(
      `[SnpDataSourceFactoryService][create()] options: ${inspect(options, { depth: 2 })}`,
    );
    switch (options.type) {
      case DsType.MYSQL:
        return new MysqlSnpDataSource(repo, options.snpColumn!);

      default:
        throw new Error(`Unsupported datasource ${options.type}`);
    }
  }
}
```
Example of initializing cache in corpdesk
```ts
export class RuntimeBootstrapService {
  static logger = new Logging();
  static svConfig = new ConfigService();
  static svSysCache = SysCacheService.getInstance(
    RuntimeCacheBootstrap.svConfig,
  );
}
```
saving data to cache
```ts
await this.svSysCache.set("runtime.role", role);
    await this.svSysCache.set("runtime.startedAt", new Date().toISOString());
```
