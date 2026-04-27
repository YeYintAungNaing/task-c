# TASK_C Automated SAST using Semgrep

This task is done on my own public github account since school github lacks features required for this task. If you want of see the all the workflows and stuff, you can see [here](https://github.com/YeYintAungNaing/task-c).

Semgrep, a static analysis engine, will be used for identifying the patterns that violate secure coding practices. Findings are exported using the static analysis result interchange format (SARIF).

The main scanning target will be the ```task_c/webapp``` but other content inside the ```task_c``` root directory including yml files are also scanned. ```.semgrepignore``` file is created to exclude database file and some local development related files since I was also scanning locally on my machine terminal.


- #### .semgrepignore file
![](./screenshots/semgrepignore.png)

----
## Pipeline architecture

- #### yml file config
![](./screenshots/workflow_yml_part1.png)

![](./screenshots/workflow_yml_part2.png)


Standardized the runner environment and installs the Semgrep CLI. Executes semgrep scan with ```p/owasp-top-ten``` rule config and two custom rules. I tried using both json and SARIF for the scan output, and decided to use SARIF since it allows GitHub to render findings directly into the security tab with a proper dynamic management interface. Jq which is a high level domain specific programming language designed for processing JSON data, is used to print out the finding summary overview from the SARIF file. ```actions/upload-artifact@v6``` and ```github/codeql-action/upload-sarif@v4``` are also used to ensure audit evidence is preserved.

---

## Custom rules

Two custom rules for hardcoded secrets and insecure cookie configuration are also added to the custom rules file. It will flag the code when a cookie is created without using ```secure=True``` and ```httponly=True``` tag. And the second rule is configured to give warning when it finds a hardcoded api key that matches the variable pattern.

- #### custom-rules.yml file
![](./screenshots/custom_rules.png)


----

## Scanning result 

The scanning pipelines fails as expected since the webapp contains a lot of vulnerabilities. 

- #### Workflow result overview
![](./screenshots/workflows.png)

----

Semgrep found 48 insecure coding practices during the scanning.
- #### Semgrep scan result overview
![](./screenshots/semgrep_summary_before_suppress.png)

---

Most of the scanning results are from the "p/owasp-top-ten" rule set while two of them are from my own custom rule. The pipeline  print outs the summarized version that is easy to read from the sarif output using ```jq``` command line.

- #### jq prints out the simple summary from SARIF file
![](./screenshots/sarif_findings_summary_before_suppress.png)

---

All the scanning results are stored inside SARIF file which can be downloaded in the artifact session. 

- #### SARIF scanning result report file
![](./screenshots/result_file.png)

The downloaded raw ```semgrep-report.sarif``` output file can be found in the root directory of this ```task_c``` folder.

---

In the security tab of the github repository, the scanning result dashboard can be found.

- #### Scanning result dashboard form GitHub
![](./screenshots/Scanning_result_dashboard_form_GitHub.png)

---

Each finding is highly detailed. The exact line, the code snippet, the rule ID as well as the rule description are shown here. 

- #### Vulnerability detail
![](./screenshots/each_vulnerability_detail.png)

---

## False Positives triage and suppression

Most the findings are legitimate findings, mainly consist of sql injection and xss related vulnerabilities. Static analysis frequently flags configurations that are technically "vulnerable" but contextually required for a development environment. 

### Hardcoded secret key

The application utilizes a non-sensitive placeholder ```'a-very-secret-and-unique-key'``` string for the local development purpose. The risk is acknowledged and mitigated by using inline # nosemgrep suppression. This also shows that my custom is massively flawed and lacks context. For example, my hardcoded rule will flag a completely normal thing such as ```key_route = '/home/key'```, since it matches the pattern of "key" and a raw string assignment.  

- #### Suppressing the rule
![](./screenshots/suppressing_the_rule_.png)

---

### Flask Debug mode 

Enabling the flask debugger is an operational requirement for the iterative development and testing phase of this project. It provides essential diagnostic features, including an interactive traceback and auto-reloading.

- #### Suppressing the rule
![](./screenshots/suppressing_the_rule.png)

---

Another scan was done after suppressing two rules and in the finding summary overview, we can see both hardcoded and debugger findings are gone, with a total of 46 findings compared to 48 in last scan.

- #### Scan result after suppressing
![](./screenshots/sarif_findings_summary_after_suppress.png)

---

## Quirky interaction when using semgrep scan together with ```---sarif```

If we look at the default semgrep summary result after suppressing (picture below), it is still saying 48 even though it should be 46 since we suppressed two false positives. Normally when we suppress using ```nosemgrep```, we can directly see the differences of finding count between scan results easily. However, when we use the ```--sarif``` flag, Semgrep is forced to obey the strict international SARIF standard. The SARIF standard states that all findings even suppressed ones must be written into the file for "auditing purposes" under a special suppressions array. If we look inside the SARIF output, we can see both of them flagged with ```suppressions``` tag correctly, which is why I was able to filter them out with ```map(select(has("suppressions")``` using ```jq``` command during the ```Vulnerability Summary``` printing workflow that I have shown in the above picture.

- #### Somewhat misleading semgrep summary after suppressing
![](./screenshots/semgrep_summary_after_suppress.png)



**Therefore, Automated SAST tools are essential for baseline enforcement but remain context-blind, and these tools should not be used as the only security gate in modern software development.**

---
















