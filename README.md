# Iac-tf

# Terraform Professional – Objective 1 Practice Labs

Oct 9, 2026 · @Casimir COMPAORE

## How to use these labs

Each lab is an end-to-end task with an outcome and acceptance criteria, not instructions. Decide the design yourself, build it, then prove it meets the "Done when" list. Labs 1c, 1d and 1e are reading labs: answer on paper first, then verify with `terraform console` or `terraform plan`, then check the answer key at the end.

**Shared scenario.** Alpenwerk GmbH, a 120-person manufacturer in Linz, is moving its customer portal to AWS. You are the TekLid consultant. All labs build pieces of the same codebase, so later labs reuse earlier work.

**Baseline environment.** Use the setup you already have: a GitHub Codespace, an HCP Terraform organization with CLI-driven workspaces, and AWS credentials supplied to HCP Terraform (dynamic credentials preferred). Region `eu-central-1` unless a lab says otherwise. Destroy what you build at the end of each session to keep costs near zero.

**Rules for yourself**

- No copying finished modules from the registry for the core labs; registry modules are allowed only where a lab says so.
- Every lab ends with a clean `terraform fmt -check`, `terraform validate`, and a plan you can explain line by line.
- Commit each lab to its own branch and write a short README explaining the design decisions. Explaining *why* is what the exam's scenario questions test.

## Lab 1a – Translate business requirements into infrastructure code

**Outcome:** a module-based codebase that deploys Alpenwerk's web tier to `dev` and `prod` from the same code, with every requirement below traceable to a specific resource, variable or validation.

**The brief you receive from the client (verbatim)**

> "We need the customer portal online in Frankfurt only — our data must stay in the EU. Production must survive the loss of one data centre. Dev can be cheap and doesn't need to be highly available. The public should only reach the site over HTTPS; our admins need shell access but security says no open SSH. Finance wants every resource tagged with cost centre CC-4410 and the environment. We'd like to add a staging environment next year without rewriting anything."

**Your deliverables**

- A requirements traceability table in the README: each sentence of the brief → the Terraform construct that satisfies it.
- A root module per environment (or one root module with per-workspace variables — justify your choice) calling child modules you write: `network`, `web`.
- Prod spans 3 AZs; dev uses 1 AZ. The AZ count must come from input, not from duplicated code.
- Public access through an ALB on 443 only, instances in private subnets, admin access via SSM Session Manager (no port 22 anywhere).
- Region and tags enforced so a teammate cannot accidentally deploy outside `eu-central-1` or forget the cost centre.

**Constraints**

- Adding `staging` must require only a new workspace and a variable file — no edits to module code.
- No hard-coded AZ names, AMI IDs, or account IDs.

**Done when**

- [ ] `dev` and `prod` plans succeed from the same module code, and their differences come only from input values.
- [ ] Setting `region = "us-east-1"` or omitting the cost centre fails at plan time with a clear message.
- [ ] `aws ec2 describe-security-groups` shows no rule allowing port 22.
- [ ] You can explain to a non-technical stakeholder, in two minutes, how each brief sentence maps to the code.

**Stretch:** add a `staging` workspace in under 10 minutes to prove the claim.

## Lab 1b – Predictable and secure Terraform configuration

**Outcome:** harden the Lab 1a codebase so that two engineers running it a month apart get the same result, bad input is rejected before anything is created, and no secret ever appears in plan output, logs, or state.

**New requirements from Alpenwerk's security officer**

- The portal gets a PostgreSQL RDS instance. Its master password must never be stored in Terraform state or shown in any plan.
- The production database and its subnet group must be impossible to destroy by accident.
- Instance types are limited to an approved list; CIDR inputs must be valid; environment names must be lowercase and one of `dev`, `staging`, `prod`.
- After apply, the team wants an automatic warning if the ALB's HTTPS listener ever stops returning HTTP 200 — without failing the run.

**Constraints**

- Pin Terraform and every provider so upgrades are deliberate; the lock file must be committed and cover the platforms your team uses (Linux in Codespaces, macOS laptops).
- Use the newest mechanisms your Terraform version supports for keeping secrets out of state (investigate ephemeral values and write-only arguments, and decide what fits).
- Use `precondition` / `postcondition` where an assumption about another resource matters (for example, that the chosen AMI is `x86_64` and the subnet is private).

**Done when**

- [ ] `grep -i password` on a downloaded state file returns nothing.
- [ ] Each invalid input you can think of (uppercase env, `t2.2xlarge`, `10.0.0.0/33`) fails with a custom error message during plan.
- [ ] A plan that would destroy the prod database is refused by Terraform itself, not by a reviewer.
- [ ] A `check` block reports a warning when the health endpoint fails, and the run still succeeds.
- [ ] Running `terraform init -upgrade` is the only way provider versions change, and you can show that in Git history.

**Stretch:** add `tflint` and `checkov` (or `trivy config`) to a GitHub Actions workflow that runs on every pull request, and fix every high-severity finding.

## Lab 1c – Trace variables, locals and outputs to a root cause

**Outcome:** find every defect in the configuration below by reasoning from symptom to source. Write your diagnosis first, then confirm it with `terraform console`, `terraform plan`, and `terraform output`.

**The ticket from Alpenwerk ops:** "We deployed prod with `environment = \"Prod\"`. Four things are wrong: (1) prod got a `t3.micro` instead of `m6i.large`, (2) prod is in only one AZ, (3) every resource is tagged `Env = legacy`, (4) the instance is named `alpenwerk-Prod-web` but our naming standard is all lowercase. Also, the `public_subnet_ids` output feeds our ALB module and the ALB keeps failing."

```hcl
variable "project" {
  type    = string
  default = "alpenwerk"
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "az_count" {
  type    = number
  default = 3
}

variable "extra_tags" {
  type    = map(string)
  default = { Env = "legacy", Owner = "ops" }
}

data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  env         = lower(var.environment)
  name_prefix = "${var.project}-${local.env}"
  is_prod     = var.environment == "prod"
  azs         = slice(data.aws_availability_zones.available.names, 0, local.is_prod ? var.az_count : 1)
  common_tags = merge({ Project = var.project, Env = local.env }, var.extra_tags)
}

resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)
  availability_zone = local.azs[count.index]
  tags              = local.common_tags
}

resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index + 10)
  availability_zone = local.azs[count.index]
  tags              = local.common_tags
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.is_prod ? "m6i.large" : "t3.micro"
  subnet_id     = aws_subnet.private[0].id
  tags          = merge(local.common_tags, { Name = "${var.project}-${var.environment}-web" })
}

output "public_subnet_ids" {
  value = [for s in aws_subnet.private : s.id]
}

output "name_prefix" {
  value = local.name_prefix
}
```

**Questions**

1. For each of the five symptoms, name the exact line that causes it and why.
2. The `name_prefix` output shows `alpenwerk-prod`. Why does that mislead someone debugging symptom 4?
3. If someone sets `az_count = 4` in a region with 3 AZs and `environment = "prod"` (after your fix), what happens at plan time? How would you make the configuration fail with a clear message instead, or degrade gracefully?
4. A workspace variable in HCP Terraform sets `environment = "prod"`, and an `env.auto.tfvars` committed in the repo sets `environment = "dev"`. Determine which value wins on a remote run and prove it with an experiment, not documentation alone.
5. Rewrite the `locals` block so this class of bug (mixing raw input with normalized values) cannot recur.

**Done when**

- [ ] All five symptoms are fixed, and a prod plan shows 3 AZs, `m6i.large`, `Env = prod`, and a lowercase name.
- [ ] Your README lists each root cause in one sentence.

## Lab 1d – Trace dynamic resources created with for\_each

**Outcome:** predict, without running anything, exactly which resource instances this configuration creates and how the plan changes under edits. Then refactor the Lab 1c network from `count` to `for_each` with zero resources destroyed.

```hcl
variable "subnets" {
  type = map(object({
    cidr = string
    az   = string
    tier = string
  }))
  default = {
    "web-a" = { cidr = "10.0.1.0/24",  az = "eu-central-1a", tier = "public" }
    "web-b" = { cidr = "10.0.2.0/24",  az = "eu-central-1b", tier = "public" }
    "app-a" = { cidr = "10.0.11.0/24", az = "eu-central-1a", tier = "private" }
    "app-b" = { cidr = "10.0.12.0/24", az = "eu-central-1b", tier = "private" }
  }
}

resource "aws_subnet" "this" {
  for_each                = var.subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = each.value.tier == "public"
  tags                    = { Name = each.key, Tier = each.value.tier }
}

resource "aws_route_table_association" "public" {
  for_each       = { for k, s in aws_subnet.this : k => s if var.subnets[k].tier == "public" }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_nat_gateway" "this" {
  for_each      = toset([for k, v in var.subnets : v.az if v.tier == "public"])
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = "???"
}
```

**Predict, then verify**

1. List every `aws_subnet` instance address exactly as `terraform state list` would print it, in order.
2. What is the address of the route table association for `web-b`, and what does `keys(aws_route_table_association.public)` return?
3. What are the keys of `aws_nat_gateway.this`, and what is `each.value` inside that resource?
4. Replace `"???"` with an expression that picks the public subnet in the NAT gateway's AZ. It must return a single string, and fail loudly if an AZ ever has two public subnets.
5. Write an output `private_subnet_ids` (a list) and an output `subnet_ids_by_name` (a map). In what order do the list elements come out, and why?
6. You delete `web-a` from the map. List every instance the plan destroys. Now imagine the same network built with `count` over a list where `web-a` is element 0. What would the plan do instead?
7. You rename the key `app-a` to `app-1a`. What does the plan propose, and how do you rename it with no infrastructure change?
8. A colleague writes `for_each = { for s in aws_subnet.this : s.id => s }` in a new resource. Predict the error and explain the rule behind it.

**Build task**

Migrate your Lab 1c `count`-based subnets to this `for_each` design on a deployed environment.

**Done when**

- [ ] The migration plan shows `0 to add, 0 to change, 0 to destroy` (moves only).
- [ ] You can explain when `count` is still the better choice.

## Lab 1e – Predict the value of expressions that use functions

**Outcome:** 24 of 26 correct on the first attempt, without `terraform console`. Write each prediction in the table, then verify every one in `terraform console` and note why you missed any. Assume `local.cfg = { tags = {} }`.

| # | Expression | Your prediction |
| --- | --- | --- |
| 1 | `cidrsubnet("10.0.0.0/16", 8, 2)` |  |
| 2 | `cidrsubnet("10.0.0.0/16", 4, 1)` |  |
| 3 | `[for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i + 10)]` |  |
| 4 | `cidrsubnets("10.0.0.0/16", 8, 8, 4)` |  |
| 5 | `cidrhost("10.0.1.0/24", 5)` |  |
| 6 | `merge({ a = 1, b = 2 }, { b = 3, c = 4 })` |  |
| 7 | `lookup({ dev = "t3.micro" }, "prod", "t3.small")` |  |
| 8 | `coalesce("", "x")` |  |
| 9 | `try(local.cfg.tags.owner, "unassigned")` |  |
| 10 | `element(["a", "b", "c"], 4)` |  |
| 11 | `slice(["a", "b", "c", "d"], 1, 3)` |  |
| 12 | `flatten([["a"], [], ["b", ["c"]]])` |  |
| 13 | `zipmap(["a", "b"], [1, 2])` |  |
| 14 | `join("-", compact(["web", "", "prod"]))` |  |
| 15 | `format("%s-%03d", "web", 7)` |  |
| 16 | `toset(["b", "a", "b"])` |  |
| 17 | `keys({ z = 1, a = 2 })` |  |
| 18 | `upper(substr("terraform", 0, 4))` |  |
| 19 | `setproduct(["dev", "prod"], ["a", "b"])` |  |
| 20 | `one([])` |  |
| 21 | `can(regex("^ami-", "ami-0abc"))` |  |
| 22 | `jsonencode({ b = 1, a = "x" })` |  |
| 23 | `{ for k, v in { a = 1, b = 2, c = 3 } : k => v * 10 if v > 1 }` |  |
| 24 | `max([3, 7, 2]...)` |  |
| 25 | `index(["a", "b", "c"], "b")` |  |
| 26 | `split(",", "a,,b")` |  |

**Then apply it:** using only functions and expressions, generate the Lab 1d `subnets` map from just `var.vpc_cidr` and `var.azs` (a list of AZ names). Public subnets take `netnum` 1, 2, 3… and private subnets 11, 12, 13…. Your expression must produce exactly the Lab 1d default map when given `10.0.0.0/16` and two AZs.

## Lab 1f – Implement advanced configurations

**Outcome:** turn the `web` module into a reusable, versioned module that other TekLid clients could consume, using the advanced language features the exam expects you to choose between confidently.

**Requirements for the module interface**

- A single object variable `web` describing the tier, where most attributes are optional with sensible defaults (`instance_type`, `min_size`, `max_size`, `health_check_path`, `extra_ingress`). A caller passing only `{ name = "portal" }` must get a working tier.
- `extra_ingress` is a list of rules; the module renders them into the security group without one hard-coded block per rule.
- An optional CloudWatch alarm set, created only when the caller enables it. The disabled case must create zero alarm resources and produce no errors in outputs.
- The launch template's user data comes from a template file with variables injected, not an inline heredoc.

**Lifecycle and refactoring requirements**

- Replacing the launch template must never cause downtime on the Auto Scaling group.
- The ASG's `desired_capacity` is managed by scaling policies; Terraform must stop reverting it.
- When the AMI changes, instances must roll, even though no instance argument references the AMI directly.
- An existing security group that ops created by hand in the console must come under Terraform management through configuration, not CLI commands.
- A resource you no longer want Terraform to manage (but must keep running) must leave state cleanly.
- A resource you rename inside the module must not be recreated for existing callers.

**Constraints**

- Decide for each requirement which construct you used (`optional()`, `dynamic`, conditional `count` / `for_each`, `templatefile`, `lifecycle` arguments, `replace_triggered_by`, `terraform_data`, `import`, `removed`, `moved` blocks) and justify it in the README.
- Publish the module to your HCP Terraform private registry with a semantic version tag, and consume it from the Lab 1a root modules with a version constraint.

**Done when**

- [ ] A minimal caller and a fully customized caller both plan cleanly.
- [ ] Changing the AMI produces a rolling replacement with no window where zero healthy targets exist.
- [ ] The hand-made security group is imported, and a following plan is empty.
- [ ] Upgrading the module from `1.0.0` to `1.1.0` (with a rename inside) produces no destroys.

## Lab 1g – Configure Terraform providers

**Outcome:** extend the portal with a CloudFront distribution and a central logging account, which forces you to work with multiple regions, multiple accounts, and modules that need more than one provider configuration.

**New requirements**

- CloudFront in front of the ALB, with a custom domain and an ACM certificate. CloudFront only accepts certificates from `us-east-1`, while everything else stays in `eu-central-1`.
- ALB and CloudFront access logs go to an S3 bucket in a separate AWS account (`logging`), reached by assuming a role from the workload account.
- Every resource in every account and region gets the cost-centre and environment tags without repeating them on each resource.
- A reusable `edge` module creates both the certificate and the distribution. It must not configure providers itself; callers decide which regions and accounts it uses.
- HCP Terraform authenticates to both accounts with dynamic (OIDC) credentials — no static keys in any workspace.

**Constraints**

- Provider version constraints follow your Lab 1b pinning policy, and the module declares the minimum it supports, not an exact version.
- No credentials, role ARNs, or account IDs hard-coded in module code.
- Add one non-AWS provider that genuinely fits the scenario (for example, DNS records at the client's DNS host, or `random`/`tls` for generated values) and explain the source address and version you chose.

**Done when**

- [ ] `terraform providers` shows every provider, alias and module requirement, and you can explain each line.
- [ ] The certificate is in `us-east-1`, the distribution uses it, and the log bucket lives in the logging account.
- [ ] Calling the `edge` module without passing the required aliased provider fails with a clear error.
- [ ] Tags appear on resources in both accounts and both regions, without a `tags` argument on most resources.

**Stretch:** explain, in your README, what changes if the `edge` module is also used by a client that has a single account and a single region.

## Lab 1h – Use the Terraform MCP server with AI coding agents

**Outcome:** a working AI-assisted workflow in your Codespace where a coding agent grounds its Terraform answers in live registry and HCP Terraform data through HashiCorp's Terraform MCP server — and a written judgement of where it helps and where it must not be trusted.

**Setup goal (no recipe given)**

Run HashiCorp's Terraform MCP server (see the `hashicorp/terraform-mcp-server` repository for current install options and toolsets) and connect it to an agent you use, such as Claude Code or VS Code agent mode, inside your Codespace. Enable the registry tools first; enable the HCP Terraform tools only after you have decided how the token is scoped and stored.

**Tasks**

1. Ask the agent to add an AWS WAF web ACL to the Lab 1g CloudFront distribution using the provider version you have pinned. Check in the agent's trace that it fetched provider documentation through the MCP server rather than answering from memory, and compare the result with the registry docs yourself.
2. Ask the agent to find a public registry module for VPC flow logs, compare it with writing your own, and recommend one. Judge the recommendation against your Lab 1b security requirements.
3. With the HCP Terraform tools enabled, have the agent list your workspaces and summarize the last run of the `prod` workspace. Decide which operations you will allow the agent to perform and which stay human-only.
4. Give the agent the Lab 1c buggy configuration and ask it to find the bugs. Compare its findings with yours.

**Constraints**

- The HCP Terraform token used by the MCP server has the least privilege that still allows task 3, is stored as a Codespaces secret, and never appears in a committed file.
- No agent may trigger an apply. Every agent-written change goes through your normal plan review and pull request.

**Done when**

- [ ] The agent's WAF code passes `terraform validate` and plans against the pinned provider version without edits, or you can name exactly what it got wrong.
- [ ] Your README has a short "AI usage policy" for TekLid engagements: allowed tools, token scope, review rules, and two failure modes you observed.

## Capstone – Onboard a second client in one day

**Outcome:** prove the whole objective at once. A second TekLid client, an Abidjan-based logistics firm, wants the same portal pattern, under time pressure, with different rules.

**Their brief:** single AWS account, region `eu-west-3` (Paris, close to their users), two environments, HTTP allowed on `dev` only, budget-sensitive (no NAT gateway in `dev`), French-language tag values, and their own DNS provider.

**Rules**

- Reuse your published modules from Labs 1f and 1g without changing their code. Where they can't meet a requirement, cut a new module version in a backward-compatible way and show that Alpenwerk's plans stay empty.
- Use the MCP-enabled agent from Lab 1h for at least one piece, and record what you accepted and what you rejected.
- Time-box it to one working day, then write a one-page retrospective: what was reusable, which requirement forced a module change, and which decision you'd make differently.

**Done when**

- [ ] Both clients deploy from the same module versions or compatible ones, and Alpenwerk's plans show no changes.
- [ ] Every requirement in the new brief is traceable in a 1a-style table.
- [ ] You can answer, without notes: "Why did you use `for_each` here, `optional()` there, and a provider alias in this module?"

## Answer key (labs 1c, 1d, 1e)

Check these only after you have written your own answers and verified them in `terraform console`.

### Lab 1c

1. **Symptoms 1 and 2 (t3.micro, one AZ):** `is_prod = var.environment == "prod"` compares the raw input `"Prod"`, so it is `false`. Use `local.env`. **Symptom 3 (Env = legacy):** in `merge()`, later arguments win, so `var.extra_tags` overwrites the mandatory `Env` tag. Put mandatory tags last. **Symptom 4 (uppercase name):** the instance `Name` tag rebuilds the name from `var.environment` instead of using `local.name_prefix`. **Symptom 5 (ALB failing):** `public_subnet_ids` iterates over `aws_subnet.private`.
2. `name_prefix` is built correctly from `local.env`, so it looks as if naming works. The bug is that the instance doesn't use it.
3. `slice()` fails at plan time because the end index (4) is greater than the list length (3). Fail clearly with a `precondition` (or a `check`) comparing `var.az_count` with the number of available AZs, or degrade gracefully with `min(var.az_count, length(...names))`.
4. Record what your experiment shows; don't rely on this key. The lesson is that values committed in `*.auto.tfvars` files compete with workspace variables, so keep each environment-specific value in exactly one place.
5. Normalize once in `locals`, add `validation` on `var.environment`, and reference only `local.*` everywhere below the `locals` block.

### Lab 1d

1. `aws_subnet.this["app-a"]`, `aws_subnet.this["app-b"]`, `aws_subnet.this["web-a"]`, `aws_subnet.this["web-b"]` — map keys sort lexically.
2. `aws_route_table_association.public["web-b"]`; `keys(...)` returns `["web-a", "web-b"]`.
3. Keys `"eu-central-1a"` and `"eu-central-1b"`. For a set, `each.value` equals `each.key`.
4. `one([for k, s in aws_subnet.this : s.id if s.availability_zone == each.key && var.subnets[k].tier == "public"])` — `one()` errors if the list has more than one element.
5. `[for k, s in aws_subnet.this : s.id if var.subnets[k].tier == "private"]` and `{ for k, s in aws_subnet.this : k => s.id }`. The list follows lexical key order: `app-a`, then `app-b`.
6. With `for_each`: `aws_subnet.this["web-a"]`, `aws_route_table_association.public["web-a"]`, and `aws_nat_gateway.this["eu-central-1a"]` (plus its EIP if keyed by AZ). With `count`: every later element shifts index, so the remaining subnets are replaced because their CIDR and AZ change at each index.
7. Destroy `app-a` and create `app-1a`. Add `moved { from = aws_subnet.this["app-a"]  to = aws_subnet.this["app-1a"] }`.
8. "Invalid for\_each argument": `for_each` keys must be known at plan time, and subnet IDs are known only after apply. Values may be unknown; keys may not.

### Lab 1e

| # | Answer | Why it trips people |
| --- | --- | --- |
| 1 | `"10.0.2.0/24"` | Adds 8 bits; netnum 2 |
| 2 | `"10.0.16.0/20"` | Each /20 step is 16 in the third octet |
| 3 | `["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]` | `range(3)` starts at 0 |
| 4 | `["10.0.0.0/24", "10.0.1.0/24", "10.0.16.0/20"]` | The /20 aligns to its own boundary |
| 5 | `"10.0.1.5"` | Host number, not an offset from .1 |
| 6 | `{ a = 1, b = 3, c = 4 }` | Later map wins |
| 7 | `"t3.small"` | Default used when the key is missing |
| 8 | `"x"` | `coalesce` skips empty strings |
| 9 | `"unassigned"` | Missing attribute errors are caught |
| 10 | `"b"` | `element` wraps (4 mod 3 = 1) |
| 11 | `["b", "c"]` | End index is exclusive |
| 12 | `["a", "b", "c"]` | Flattens nested lists recursively |
| 13 | `{ a = 1, b = 2 }` |  |
| 14 | `"web-prod"` | `compact` removes empty strings |
| 15 | `"web-007"` | Zero-padded to 3 digits |
| 16 | `toset(["a", "b"])` | Duplicates removed; shown sorted |
| 17 | `["a", "z"]` | Keys are returned in lexical order |
| 18 | `"TERR"` | Offset 0, length 4 |
| 19 | `[["dev","a"], ["dev","b"], ["prod","a"], ["prod","b"]]` | Cartesian product |
| 20 | `null` | Errors only for 2+ elements |
| 21 | `true` | `can` turns errors into `false` |
| 22 | `"{\"a\":\"x\",\"b\":1}"` | Object keys are sorted |
| 23 | `{ b = 20, c = 30 }` | Filter runs before the result |
| 24 | `7` | `...` expands the list into arguments |
| 25 | `1` | Zero-based |
| 26 | `["a", "", "b"]` | Empty segments are kept |

One possible solution for the "apply it" task:

```hcl
locals {
  subnets = merge(
    { for i, az in var.azs : "web-${substr(az, -1, 1)}" => { cidr = cidrsubnet(var.vpc_cidr, 8, i + 1), az = az, tier = "public" } },
    { for i, az in var.azs : "app-${substr(az, -1, 1)}" => { cidr = cidrsubnet(var.vpc_cidr, 8, i + 11), az = az, tier = "private" } }
  )
}
```
