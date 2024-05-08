PJ architecture and development rules
Adopted technology
PHP 8.1
Laravel 11.x
MySQL8
php-cs-fixer
PJ architecture
Package composition
From src/app
├── Console               // Commandクラス群（Laravel標準）
├── Domain                // ドメイン層
|   ├── Domain1           // Domain1のフォルダ
|   |   ├── Repositories  // Domain1のRepositoryインタフェース群
|   |   |   ├── Input     // Domain1のRepoInputクラス群
|   |   |   └── Output    // Domain1のRepoOutputクラス群
|   |   ├── Services      // Domain1のServiceインタフェース群
|   |   |   ├── Input     // Domain1のServInputクラス群
|   |   |   └── Output    // Domain1のServOutputクラス群
|   |   └── Vo            // Domain1のVoクラス群
|   ├── Domain2           // Domain2のフォルダ
|   ...
├── Exceptions            // 例外クラス群
├── Http                  // アプリケーション層（Laravel標準）
|   ├── Controllers       // Actionクラス群
|   ├── Requests          // Requestクラス群
|   └── Responses         // Responseクラス群
├── Infra                 // インフラ層
|   ├── Domain1           // Domain1のフォルダ
|   |   ├── Repositories  // Domain1のRepository実装クラス群
|   |   └── Services      // Domain1のService実装クラス群
|   ├── Domain2           // Domain2のフォルダ
|   ...
├── Models                // Modelクラス群（Laravel標準）
├── UseCases              // アプリケーション層から呼び出されるメインUseCaseインタフェースと実装クラス群
└── Utils                 // 共通使用するUtilityクラス群
About classes under Domain
Repository interface, RepoInput class, RepoOutput class
The Repository interface is responsible for registering, updating, deleting, and referencing a single DB table. Therefore, when implementing bulk update processing that spans multiple tables, the basic format is to call the processing of multiple Repository interfaces from the Service class or UseCase class. As a general rule, define an interface for each query and use it as a single action (defined with __invoke()).
命名規則：I****Repository
The RepoInput class is a DTO class defined as an argument of the __invoke() function of the Repository interface. If the number of argument items is small (0 to 2), it does not necessarily need to be implemented.
命名規則：****RepoInput
The RepoOutput class is a DTO class defined as the return value type of the __invoke() function of the Repository interface. It does not necessarily need to be implemented if the return value is void or a single object.
命名規則：****RepoOutput
Service interface, ServInput class, ServOutput class
The Service interface is an interface that handles updating and reference processing for multiple DB tables and a certain set of business logic. The Service interface is assumed to be called from the UseCase class described below, and the basic format is to call the necessary Repository interface processing from the Service class. As a general rule, use a single action (defined with __invoke()).
命名規則：I****Service
The ServInput class is a DTO class defined as an argument of the __invoke() function of the Service interface. If the number of argument items is small (0 to 2), it does not necessarily need to be implemented.
命名規則：****ServInput
The ServOutput class is a DTO class defined as the return value type of the __invoke() function of the Service interface. It does not necessarily need to be implemented if the return value is void or a single object.
命名規則：****ServOutput
About classes under Http
Action class
This is the so-called Controller class. However, define one class for each endpoint, and as a general rule, use a single action (defined with __invoke()).
命名規則：****Action
Request class
A class that stores request parameters defined as arguments of __invoke() of the Action class and UseCase interface. As a general rule, inherit and implement the BaseFormRequest class, which is the standard Request class. In addition to Laravel's general validation implementation, create a getter function for each parameter, and in Action and UseCase classes, parameters are basically obtained through getters.
命名規則：****Request
Response class
A class that stores response parameters, defined as the return type of __invoke() in the Action class and UseCase interface. In most cases, it is implemented by extending Illuminate\Http\Resources\Json\JsonResource. No implementation is required if the API returns 204 (No Content) as normal. Instantiation is done using the UseCase class.
命名規則：****Response
About classes under Infra
Repository interface implementation class
A class that implements specific processing of the Repository interface under Domain. Inject the relevant Model class with the constructor and implement a single query process for the target table.
命名規則：****Repository (インタフェース名から「I」を取り除いたもの)
Service interface implementation class
A class that implements specific processing of the Service interface under Domain. Inject the necessary Repository interface with the constructor and implement a series of processing for one or multiple tables. Log output should be performed at the start and end of processing. When creating a DB transaction, implement it in units of the __invoke() method of this class.
命名規則：****Service (インタフェース名から「I」を取り除いたもの)
About classes under Models
Model class
Laravel standard so-called Model class. Column names should be defined as const in the class. It is mandatory to define the $connection, $table, $primaryKey, and $guarded members. (On the other hand, in principle, defining $fillable is not necessary.) If the table is the target of partitioning (sharding), implement it accordingly (the specific implementation method of partitioning will be explained separately). Please check with the PL regarding the implementation policy for table relations (they may not implement them on purpose).
命名規則：[テーブル名単数形]Model
About classes under UseCases
UseCase class (and interface)
A class that receives a Request instance from the Action class, executes the main processing by calling the Service interface or Repository interface as appropriate, and returns a Response instance to the Action. Create individual interfaces in the same folder and then implement them. It is assumed that one UseCase class will be implemented for each Action class. Log output should be performed at the start and end of processing.
命名規則：****UseCase (インタフェース名から「I」を取り除いたもの)
About classes under Utils
Commonly used Utility classes
A group of classes that are widely used in common across layers and domains. As a general rule, processes that involve DB access or external communication should not be implemented here, but should be implemented as appropriate domain-specific processes. Create appropriate subfolders depending on the content. To avoid creating arbitrary classes, be sure to consult with your PL beforehand if you want to create a new class in this folder.
命名規則：単一のルールはなし。適宜PLやメンバーと相談し意味的整合性と統一感のある命名を行うこと。
Regarding the implementation levels of the 3 layers: UseCase, Service, and Repository
Although there are no firm rules, please implement the following policies and ideas as a general rule.
UseCase: Only calls to the Service class and Repository class and the flow of the largest branch are implemented.
Also, in this layer, it is desirable that IDs and variables that are only referenced within the infrastructure (such as those that are automatically numbered by the DB) do not appear.
Service: Calls to other Service and Repository classes, detailed calculation logic, and branching are implemented.
Repository: Performs processing in as narrow a scope as possible. Basically, one query exchange with the DB (there may be up to two queries depending on the case). It is better not to include too much calculation logic.
About the number of queries in a series of UseCase processing and the implementation unit of the Repository class
This is also a very difficult point, but please implement it using the following ideas as much as possible. (I understand that there may be exceptional cases, but please consider this as the basic policy.)
Except for list search APIs that allow you to specify complex search conditions, generally do not implement complex queries that make use of many JOINs in the Repository class.
If you only want to reference from several tables based on ID items, Service or UseCase can make full use of the function that retrieves records using the ID of each Repository class as a key, even if it is multiple queries. View and update information.
In the case of the above-mentioned search processing or complex aggregation processing, it is assumed that it would be more efficient to leave those processing to the DB, so for such processing, create a special query specifically for that service. It is also possible to (However, it is not recommended to create it indiscriminately without meaning.)
development rules
Development flow
Create a PBI branch or a working branch from the develop branch.
branchName example: feature/{backlog番号}eg)feature/ADVPRO-0000
PBI: pbi
Function creation:feature
Bug fixes:fix
Improvement: refactor *As a general rule, if issues in Backlog are parent and child, please create branches accordingly.
After completing the implementation and verifying that it works in the local environment, create a PR for the develop or PBI branch.
Notify reviewers in Backlog that you created the PR.
After responding to the comments from the reviewers, etc., and receiving approval from everyone, the person in charge should perform a merge ( Squash and merge ) and delete the work branch that is no longer needed.
If there is no other work to be done, the person in charge should close the Backlog issue at the same time as the PR close.
commit rules
You are free to express your comments in commit units, but please be aware of the following points.
Do not include "", "\" or "\" in commit (and PR) titles, as this can affect deployments and releases.
When responding to PR review issues, try to make one commit per reviewer review. When confirming modifications to reviewers, try to be able to show the corrected content in just one commit comment.
coding rules
I will list the main rules. I'm sure there are more than what's listed here, but it would be helpful if you could basically align them with existing codes and understand them in reviews and consultations.
type is required
Please write all method arguments, member variables, and return values.
Generating an instance
For classes other than those that store values ​​such as Vo, Input/Output classes, etc., please obtain an instance using constructor injection. In that case, please inject using the interface name and write the implementation class name in @param of the constructor's phpdoc.
ServiceProvider.phpDon't forget the DIBinding
How to write member variables and constructors
We recommend reducing the amount of coding by using PHP RFC: Constructor Property Promotion .
Compliant with PSR-12
Detail isphp_cs.dist.php
If you are using VSCode
Open VSCode in your workspace (steps below)
Select “File” → “Open Workspace with File” from the menu
Select "skyticket-point.code-workspace" under skyticket-point and press "Open"
name of the class
Initial capital camel case (except migrations)
Variable name
initial lowercase camel case
constant name
uppercase snake case
*Please be especially careful about capitalization errors in folder names and file names!
Please refer to the following for class names, variable names, and method names (please use names that are easy for others to understand based on the processing content, rather than names that only you can understand)
Class name edition
Variable name
Please be aware of singular and plural forms when naming variables and class names (however, this may make it more difficult to understand, so please consult us if you are unsure.)
Do not use function names such as "get~" for anything other than pure getter functions. Use "Find" etc. in the class name as well.
Implementation of setter functions and assignment to member variables outside of the constructor are prohibited.
If your editor is VSCode, be sure to include the following extensions.
   # ターミナルから実行で入ります
   code --install-extension cvergne.vscode-php-getters-setters-cv
   code --install-extension felixfbecker.php-intellisense
   code --install-extension fetzi.php-file-types
   code --install-extension MehediDracula.php-namespace-resolver
   code --install-extension neilbrayfield.php-docblocker
   code --install-extension xdebug.php-debug
   code --install-extension xdebug.php-pack
   code --install-extension zobo.php-intellisense
Please write all comments, logs, response error messages, and other messages for developers in English.
Log implementation policy
Always implement INFO log output at the start and end of UseCase class and Service class .
situation   Example message log context
Start processing    "Started ~."  ID field of main processing target,
operator ID of operator in case of management screen
Processing start end    "Finished ~." ID field of main processing target,
operator ID of operator in case of management screen
Implementation sample
   public function __invoke(XxxUpdateRequest $request): array
   {
       Log::info('Started updating a xxx.', [
           'xxxId' => $request->getXxxId(),
           'operatorId' => $request->getOperatorId(),
       ]);
       // ... main process
       Log::info('Finished updating a xxx.', [
           'xxxId' => $request->getXxxId(),
           'operatorId' => $request->getOperatorId(),
       ]);
       return new XxxUpdateResponse($xxx);
   }
Be sure to implement ERROR log output at the beginning of the catch block .
Implementation sample
   public function __invoke(XxxUpdateRequest $request): array
   {
       try {
           // ... main process
       } catch (Throwable $e) {
           # Make sure to include ID to be processed or other key information in the context if possible
           Log::error($e->getMessage(), [
               'trace' => $e->getTrace(),
               'xxxId' => $xxxId,
           ]);
           // ... exception handling
       }
   }
Exception control policy
As a general rule, except for UseCase classes, be sure to throw exceptions to the upper layer without squeezing them.
In the UseCase class, the basic policy is to throw exceptions other than some business-related exceptions to a higher level, and ultimately perform unified exception control in Exceptions/Handler.php.
However, if you are unable to rank higher due to individual circumstances, be sure to check with reviewers to see if there is a problem.

