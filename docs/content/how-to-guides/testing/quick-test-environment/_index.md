---
type: "docs"
title: "Quickly Test a Drasi Environment"
linkTitle: "Quickly Test a Drasi Environment"
weight: 10
description: >
    Learn how to quickly test a Drasi environment to make sure it works end-to-end
---

Sometimes, especially after initial installation, it is helpful to be able to quickly test that a Drasi environment works as expected. This guide describes how to run a pre-packaged quick test on a Drasi environment using a single command. When run, the quick test will:
1. Create a PostgreSQL database and loads some test data
1. Create a [PostgreSQL Source](/how-to-guides/configure-sources/configure-postgresql-source/) that connects to the test database
1. Create a Continuous Query that uses the PostgreSQL Source
1. Create a Reaction that subscribes to the Continuous Query
1. Make changes to the data in the the PostgreSQL database and verify that those changes result in the expected output from the Reaction

If the quick test is successful, it will delete the Reaction, Continuous Query, Source, and PostgreSQL database returning the Drasi environment to the state is was in before the quick test was run. 

If the quick test is unsuccessful, it will not delete anything so that you can investigate the cause of any problems using the output of the test and the logs generated by the test components.

> A detailed description of the steps included in the quick test are provided in the [Quick Test Detail](#quick-test-detail) section below.

### Prerequisites
On the computer from which you will run the quick test, you need to install the following software:
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [Drasi CLI](/reference/command-line-interface/)

You also need access to the Drasi environment that you want to test.

### Running the Quick Test
To run the quick test on a Drasi environment that is installed in the default `drasi-system` namespace, run the following command:

{{< tabpane langEqualsHeader=true >}}
{{< tab header="MacOS or VS Code Dev Container" lang="Bash" >}}
curl -fsSL https://raw.githubusercontent.com/drasi-project/drasi-platform/main/dev-tools/testing/quick-test-environment/quick-test-environment.sh | /bin/bash
{{< /tab >}}
{{< tab header="Windows PowerShell" lang="Bash" >}}
Invoke-Command -ScriptBlock ([scriptblock]::Create([System.Text.Encoding]::UTF8.GetString((New-Object Net.WebClient).DownloadData('https://raw.githubusercontent.com/drasi-project/drasi-platform/main/dev-tools/testing/quick-test-environment/quick-test-environment.ps1')))) 
{{< /tab >}}
{{< /tabpane >}}

If you need to test a Drasi environment installed in a non-default namespace, you will need to provide the namespace name as an argument to the command, as shown with the `<namespace>` placeholder in the following commands:
{{< tabpane langEqualsHeader=true >}}
{{< tab header="MacOS or VS Code Dev Container" lang="Bash" >}}
curl -fsSL https://raw.githubusercontent.com/drasi-project/drasi-platform/main/dev-tools/testing/quick-test-environment/quick-test-environment.sh | /bin/bash -s <namespace>
{{< /tab >}}
{{< tab header="Windows PowerShell" lang="Bash" >}}
Invoke-Command -ScriptBlock ([scriptblock]::Create([System.Text.Encoding]::UTF8.GetString((New-Object Net.WebClient).DownloadData('https://raw.githubusercontent.com/drasi-project/drasi-platform/main/dev-tools/testing/quick-test-environment/quick-test-environment.ps1')))) -ArgumentList <namespace>
{{< /tab >}}
{{< /tabpane >}}

### Quick Test Success
The quick test generates output to the terminal to help in problem resolution should anything go wrong. If the quick test completes successfully, you will see the following final lines of output in the terminal:

```
...
Quick test passed!
cleaning up resources...
configmap "test-data-init" deleted
configmap "test-pg-config" deleted
deployment.apps "postgres" deleted
service "postgres" deleted
✓ Delete: Source/quick-test: complete
✓ Delete: ContinuousQuery/quick-query: complete
✓ Delete: Reaction/quick-result-reaction: complete
```

As noted at the end of the quick test output, all the Drasi resources created for the quick test have now been deleted and the Drasi environment is returned to the state it was in before the test.

### Quick Test Failure
If the quick test fails, depending on the cause, you will see output similar to the following:

```
...
Inserting the following entries into the database: '{Id: 5, Name: Item 5, Category: A}'
INSERT 0 1
Retrieving the current result from the debug reaction
Final output:[{"Category":"A","Id":1,"Name":"Item 1"},{"Category":"A","Id":3,"Name":"Item 3"}]  # The result set did not update after the insertion
Expected output after the insertion:[{"Category":"A","Id":1,"Name":"Item 1"},{"Category":"A","Id":3,"Name":"Item 3"},{"Category":"A","Id":5,"Name":"Item 5"}]
Quick test failed
Resources are not deleted. If you wish to clean up everything, run 'curl -s https://raw.githubusercontent.com/drasi-project/drasi-platform/main/dev-tools/testing/quick-test-environment/cleanup-quick-test-environment.sh | bash'
```

The quick test output should give insight into what failed, but it will not always be able to see from the output the cause of the problem and how to fix it. Some ways to investigate further include:

- Looking at Drasi resource status using the [`drasi list`](/reference/command-line-interface/#drasi-list) command
- Looking at Drasi resource configuration and status using the [`drasi describe`](/reference/command-line-interface/#drasi-describe) command
- Looking at the logs generated by the running Drasi components using the `kubectl logs` command

### Quick Test Detail
It is not necessary to understand the detail of what the quick test does in order to use it; this section is only included in case you want to understand the quick test process in more detail.

> The source code for the quick test is available on GitHub in the [drasi-platform](https://github.com/drasi-project/drasi-platform) repo in the [dev-tools/testing/platform-quick-test](https://github.com/drasi-project/drasi-platform/tree/main/dev-tools/testing/platform-quick-test) folder.

This quick test script does the following:

1. Sets up a PostgreSQL pod in your Kubernetes cluster. The pod will contain one database with the name `quickdb`. The database will contain one table called `Item` with the following entries:
| id |  name  | category |
|----|--------|----------|
|  1 | Item 1 | A        |
|  2 | Item 2 | B        |
|  3 | Item 3 | A        |

2. Create a PostgreSQL Source that will connect to the PostgreSQL pod that was created in the previous step.
3. Create a Continuous Query that will observe all Items that are in Category A. It is defined with the following Cypher query:
   ```cypher
   MATCH
      (i:Item {category: 'A'})
    RETURN
      i.id as Id,
      i.name as Name,
      i.category as Category
   ```
4. Create a Result Reaction to monitor the Continuous Query's result set.
5. After all Drasi resources are created, the quick test will use the Result Reaction to validate the Continuous Query's initial result set, which should contain only these two entries:
    ```json
    {"Category":"A","Id":1,"Name":"Item 1"},
    {"Category":"A","Id":3,"Name":"Item 3"}
    ```
6. The quick test will then insert the following two entries into the Item table of the the PostgreSQL database:
    ```json
    {"Id": 4, "Name": "Item 4", "Category": "B"}
    {"Id": 5, "Name": "Item 5", "Category": "A"}
    ``` 
7. The quick test then verifies the new database inserts resulted in the expected output from the Result Reaction. Only the Item with an ID of `5` is in category A. As a result, we should expect the final Continuous Query result set to be: 
    ```json
    {"Category":"A","Id":1,"Name":"Item 1"},
    {"Category":"A","Id":3,"Name":"Item 3"},
    {"Category":"A","Id":5,"Name":"Item 5"}
    ```
8. Finally, the quick test cleans-up by deleting all of the components it created.