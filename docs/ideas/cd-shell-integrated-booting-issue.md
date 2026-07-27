

cd-node project for corpdesk has now merged cd-cli, cd-rpc and cd-api.
This is possible because right from the design, the distincion between each sub-system was meant to be the booting process.
The runtime uses samed code base that are arranged in 'app' and 'sys' directory. Inside any of the two is same structure. list of module directories each with 'controllers', 'models' and 'services'. And the coding standards are the same.
We are in the process for merging cd-shell.
The main issue is arrising during build where the builder is rejecting server modules that are not compatible with browser modules.
Without looking at a lot of details, I just want us to focus on specific issue displayed in the logs and the shared code for ssh.service.ts.
My plan is to identify the problematic modules eg node-ssh shown in this demo.
For each problematic issue, we make them load dynamically in a way that during the build, they are not on the way.
But during runtime, we have a context (ICdExecutionContext) and applicatin descriptor(CdAppDescriptor) information that has active environmental parameters, which can be used to enable a given module. Note that descriptor information can be used to tell the dependencies of any file.
For example we can make the dynamic load for node-ssh to be conditional so that when role:CdNodeRole===CdNodeRole.CD_SHELL it should not be possible to load.

Much as I have done a lot of context explanation, I just need you to confirm whether a plan like this can work?
Can you give me a guide on how to make the ssh.service.ts depand on a dynamic loading that can be appropirate for this case.


```log
emp-12@emp-12 ~/cd-node (main)> npm run shell-build

> cd-node@0.0.1 shell-build
> npm run shell-clean && npm run shell-prebuild && npm run shell-compile-ts && vite build && npm run shell-build-mdc && npm run shell-post-build


> cd-node@0.0.1 shell-clean
> rm -rf dist dist-ts


> cd-node@0.0.1 shell-prebuild
> node scripts/prebuild-stubs.js

[prebuild] View placeholders ready.


> cd-node@0.0.1 shell-compile-ts
> tsc --project tsconfig.json

vite v5.4.21 building for production...
✓ 1010 modules transformed.
x Build failed in 4.10s
error during build:
[commonjs] node_modules/ssh2/lib/protocol/crypto/build/Release/sshcrypto.node (1:0): Unexpected character '\u{7f}' (Note that you need plugins to import files that are not JavaScript)
file: /home/emp-12/cd-node/node_modules/ssh2/lib/utils.js:1:0

1: ELF>@�P@8
            @xAxA...
   ^
2:  
3:  

    at getRollupError (file:///home/emp-12/cd-node/node_modules/rollup/dist/es/shared/parseAst.js:317:41)
    at ParseError.initialise (file:///home/emp-12/cd-node/node_modules/rollup/dist/es/shared/node-entry.js:14814:28)
    at convertNode (file:///home/emp-12/cd-node/node_modules/rollup/dist/es/shared/node-entry.js:16790:10)
    at convertProgram (file:///home/emp-12/cd-node/node_modules/rollup/dist/es/shared/node-entry.js:16026:12)
    at Module.setSource (file:///home/emp-12/cd-node/node_modules/rollup/dist/es/shared/node-entry.js:17769:24)
    at async ModuleLoader.addModuleSource (file:///home/emp-12/cd-node/node_modules/rollup/dist/es/shared/node-entry.js:21946:13)
emp-12@emp-12 ~/cd-node (main) [1]> 
```

---

This is happending due to this file:

```ts
// src/CdNode/sys/dev-descriptor/services/ssh.service.ts

/* eslint-disable style/operator-linebreak */
/* eslint-disable style/brace-style */
import { NodeSSH } from "node-ssh";

export class SshService {
  private ssh = new NodeSSH();
  // private ssh: any = {};


  constructor() {
  }
  /**
   * Establishes an SSH connection and executes a command.
   */
  async executeCommand(
    config: {
      host: string;
      username: string;
      privateKey?: string;
      password?: string;
    },
    command: string,
  ): Promise<string> {
    try {
      await this.ssh.connect({
        host: config.host,
        username: config.username,
        privateKey: config.privateKey,
        password: config.password,
      });

      const result = await this.ssh.execCommand(command);
      this.ssh.dispose();
      return result.stdout || result.stderr;
    } catch (error) {
      throw new Error(`SSH execution failed: ${(error as Error).message}`);
    }
  }

  requiresSSH(
    accessScope: string,
    physicalAccess: string,
    transport?: { protocol: string },
  ): boolean {
    return (
      accessScope === 'remote' &&
      (physicalAccess !== 'direct' || transport?.protocol === 'ssh')
    );
  }
}

```

```ts
export interface ICdExecutionContext {
  //
  // Identity
  //

  identity: {
    requestId?: string;

    executionId: string;

    role: ICdNodeRole;

    // transport: "cli" | "rpc" | "http" | "grpc";
  };

  //
  // Request
  //

  request?: ICdRequest;

  response?: ICdResponse;

  //
  // Organism
  //

  organism: {
    initializedAt: Date;

    projectRoot?: string;

    profiles?: ProfileContainer;

    profileLoaded: boolean;

    aiReady: boolean;

    workflowReady: boolean;

    cacheReady: boolean;

    snpReady: boolean;
  };

  //
  // Runtime
  //

  runtime?: {
    workflow?: any;
    descriptor?: any;
    cache?: Map<string, any>;
    startedAt?: number;
    completedAt?: number;
    duration?: number;
  };

  //
  // Execution
  //

  execution: {
    status?: "created" | "running" | "completed" | "failed";
    errors?: string[];

    warnings: string[];
  };

  // Snp

  snp?: {
    adapters?: string[];

    resources?: string[];
  };

  //
  // Metadata
  //

  meta?: {
    controller?: string;

    action?: string;

    module?: string;

    userId?: number;

    traceId?: string;
  };
}
```

```ts
export interface CdAppDescriptor extends BaseDescriptor {
  $schema?: string;
  name: string;
  type?: AppType;
  projectGuid?: string;
  parentProjectGuid: string | null;
  modules: CdModuleDescriptor[];
  cdCi?: CiCdDescriptor;
  description?: string;
  language?: LanguageDescriptor; // getLanguageByName(name: string,languages: LanguageDescriptor[],)
  environments?: EnvironmentDescriptor[]; // Development environment settings
  versionControl?: VersionControlDescriptor; // Version control details
  directorySignature?: DirectorySignatureDescriptor;
}
```