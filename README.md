# Test framework for Unit tests in Dyalog APL


`Tester` is a member of the APLTree library. The library is a collection of classes etc. that aim to support the Dyalog APL programmer. Search GitHub for "apltree" and you will find solutions to many every-day problems Dyalog APL programmers might have to solve.


## Overview 

The purpose of this class is to provide a framework for testing all the projects of the APLTree library. Only with such a framework is it possible to make changes to any APLTree project with confidence.

You might find the framework flexible enough to suit your own needs when it comes to implementing tests, even independently from the APLTree project.


## Details 


### Terminology

Note that test cases causing a crash are considered "broken". Test cases that do not return the expected result are considered "failing".


### Assumptions and preconditions

1. The `#.Tester` class assumes that all your tests are hosted by a namespace. It must not be an unnamed namespace. All methods of the `Tester` class running test cases take a ref to that namespace as an argument.

1. All test functions inside that namespace are expected to start their names with the string `Test_` followed by digits. '''Update''': since version 1.1 test functions can be grouped. For example, in order to group all functions testing a method `foo` the test functions are named `Test_foo_001`, `Test_foo_002`, `Test_foo_003` and so on.

1. The number of digits you use for numbering is not restricted: `Test_foo_1` is as fine as is `Test_foo_0000001`. However, they should be consistent. When starting to implement test cases you are advised to leave gaps: `Test_foo_001`, `Test_foo_010` etc.

1. The first line after the header must carry a comment telling what is actually tested. Keep in mind that later this is the only way to tell one test case from the others without reading the code, so be as clear as you can possibly be.

1. It is assumed that you have three different scenarios when you want to run test cases with a different behaviour:
    
   * Run with error trapping and return a log (vector of text vectors) reporting details. Run this before checking your code into a Code Management System like SubVersion, GitHub or [acre](https://github.com/the-carlisle-group/Acre-Desktop). See `Tester.Run` for details.
   
   * Run without error trapping. Use this for investigating why test cases crash or fail. See `Tester.RunDebug` for details.
   
   * Run "batch mode". It is assumed that those test cases that need a human being in front of the monitor are '''not''' executed when running in batch mode. See `Tester.RunBatchTests` for details. (Of course it is best not to rely on a human being but this cannot always be avoided). There is also a method `RunBatchTestsInDebugMode` available.

1. Every test function must accept a right argument which is a two-item vector of Booleans:

   1. `debugFlag` (0=use error trapping)
   1. `batchFlag` (0=does not run in batch mode)

1. Every test function must return a result. Use one of the following niladic functions (acting like constants) as explicit result:

|-----------------------|-----------------------------------------------------------------------------|
| `∆OK`                 | Passed |
| `∆Failed`             | Unexpected result|
| `∆NoBatchTest`        | Not executed because `batchFlag` was 1.|
| `∆InActive`           | Not executed because the test case is inactive (not ready, buggy, whatever) |
| `∆LinuxOnly`          | Not executed because runs under Linux only.|
| `∆LinuxOrMacOnly`     | Not executed because runs under Linux/Mac OS only.|
| `∆LinuxOrWindowsOnly` | Not executed because runs under Linux/Windows only.|
| `∆MacOrWindowsOnly`   | Not executed because runs under Mac OS/Windows only.|
| `∆MacOnly`            | Not executed because runs under Mac OS only.|
| `∆NoAcreTests`        | Not executed because it's acre related.|
| `∆WindowsOnly`        | Not executed because runs under Windows only.|

These niladic functions are made available as Helpers - see there.


## Work flow

No matter which of the `Run*` functions (`Run`, `RunBatchTests`, `RunDebug`, `RunThese`) you are going to call, the workflow is always the same:


### INI files

First of all the `Run*` method checks whether there is a file `testcases_{computername}.ini`. If this is the case that INI file is processed. Use this to specify computer-dependent variables.

If no computer-specific INI file was found then it checks whether there is a file `testcases.ini`. If this is the case that INI file is processed, too. Use this to specify general stuff that does not depend on a certain computer/environment.

Note that if one of the two INI files exists there will be a flat namespace `{yourTestNamespace}.INI`, meaning that sections in the INI file are ignored. An example: if your test functions are hosted by a namespace `Foo` and your INI file specifies a variable `hello` as holding "world" then:

```
       'world'≡#.Foo.INI.hello
1
```


### Initialisation

Now the `Run*` method checks whether there is a function `Initial` in the hosting namespace. If that is the case then the function is executed.

Note that the function must be either niladic or monadic and --(must not return a result)-- and, since version 3.0, may or may not return a Boolean result. A 1 means that function did what it is supposed to do (=same as no result) while a 0 means it could not initialize.

Of course you can simply execute `→` on a single line in your `Initial` function if any requirement is not met but that would also mean that your test did not run at all. If you run your test case automatically somehow than it should return a 1 indicating failure. Also, part of the initialization might have been carried out, and a function `Cleanup` might get rid of any left-overs.

If the function is monadic then a Boolean is passed as the right argument telling the `Initial` function whether the test suite is going to run in batch mode (1) or not.

Use this function to create stuff that's needed by '''all''' test cases, or tell the user something important - of course only if the batch flag is false.

It might  be a good idea to call a function tidying up in line 1, just in case a failing test case has left something behind; see below for details.

Another thing to mention is that `Initial` is not a bad place to establish the Helpers - see there for details.


### Finally: running the test cases

Now the test cases are executed one by one.


### Tidying up

After the last test case was executed the `Run*` function checks whether there is a function `Cleanup` in the namespace hosting your test cases. If that's the case then this function is executed. Such a function must be a niladic, no-result returning function.


### INI file again

Finally the namespace "INI" holding variables populated from your INI file(s) is deleted.


## Helpers

Over time it emerged that certain helpers are very useful in the namespace hosting the test cases. Therefore with version 1.4.0 a method `EstablishHelpersIn` was introduced that takes a reference as the right argument. That ref should point to the namespace hosting the test cases. If you are executing `#.Testers.EstablishHelpersIn` from within that namespace you can specify an empty vector as the right argument: it then defaults to `⎕IO⊃⎕RSI`.

One of the helpers is the method `ListHelpers` which returns a table with all names in the first column and the first line of the function (which is expected to be a comment telling what the function actually does) in the second column:

```
      ListHelpers
 Run                       Run all test cases                                                           
 RunDebug                  Run all test cases with DEBUG flag on                                        
 RunThese                  Run just the specified tests.                                                
 RunBatchTests             Run all batch tests                                                          
 RunBatchTestsInDebugMode  Run all batch tests in debug mode (no error trapping) and with stopFlag←1.   
 E                         Get all functions into the editor starting their names with "Test_".         
 L                         Prints a list with all test cases and the first comment line to the session. 
 G                         Prints all groups to the session.                                            
 FailsIf                   Usage : →FailsIf x, where x is a boolean scalar                              
 PassesIf                  Usage : →PassesIf x, where x is a boolean scalar        
 GetTestFnsTemplate        ⍝ Returns a vector of text vectors with the code of a template test case.                     
 GoToTidyUp                Returns either an empty vector or "Label" which defaults to ∆TidyUp          
 RenameTestFnsTo           Renames a test function and tell acre.                                       
 ListHelpers               Lists all helpers available from the `Tester` class.                         
 ∆...                      ⍝ Niladic functions returning an integer constant, see table above.
```

The helpers fall into four groups:


### Running test cases

`Run`, `RunDebug`, `RunThese`, `RunBatchTest` and `RunBatchTestsInDebugMode` are running all or selected test cases with or without error trapping. They are shortcuts for calling the methods in `Tester`.


### Flow control

`FailsIf`, `PassesIf` and `GoToTidyUp` control the flow in test functions.

These functions return a result (Boolean) in case error trapping is active but make the calling `Test*` crash otherwise, allowing you to investigate a failing test case right on the spot.

This is achieved by the functions `FailsIf` and `PassesIf` signalling an error that can be trapped with:

```
      ⎕TRAP←(999 'C' '. ⍝ Deliberate error')(0 'N')
```

That's why the template for the test function carries such a statement '''and''' keeps `⎕TRAP` local.

Note that `GoToTidyUp` allow you with a statement like

```
 →GoToTidyUp expected≢result
```

to jump to a label `∆TidyUp`. This is useful in case a test case needs to do some cleaning up.


### Test function template

In case the namespace has no test functions in it (yet) the `EstablishHelpersIn` function also creates a test function template named `Test_000`. ('''Not''' when the hosting namespace is scripted!)


### "Constants"

These niladic functions behave like constants and are useful for assigning explicit results in test functions.


### Misc

`E`, `G` and `L` allow the user to edit all test functions and to list test groups, if any.


### Examples

The test template contains examples for how to use the flow control functions.

The `Run*` functions are discussed later.

The most important helper functions are probably `G` and `L`: with a large number of test functions it's the only reasonable way to gather information: `G` lists all groups and `L` all test functions.

Since version 1.8.0 `L` requires a right argument (empty or group name) and accepts an optional left argument (test case numbers).

Examples from the `Fire` development workspace:

```
      ⍴L ''
99 150
      G
acre           
InternalMethods
List           
Misc           
Replace        
ReportGhosts   
Search         
      L 'Li'
Test_List_001             Unnamed NSs & GUI instances are ignored.                                        
Test_List_002             Just GUI instances are ignored                                                  
Test_List_003             Nothing is ignored at all; instances of GUI objects are treaded like namespaces 
Test_List_004             Nothing is ignored but class instances                                          
Test_List_005             Scan all object types but with "Recursive"≡None                                 
Test_List_006             Scan all object types but with "Recursive"≡Just one level                       
Test_List_007             Search for "Hello" with a ref and an unnamed namespace                          
    1 7 99 L 'List'
Test_List_001             Unnamed NSs & GUI instances are ignored.               
Test_List_007             Search for "Hello" with a ref and an unnamed namespace   
```

Note that you don't need to specify more characters than needed to actually identify a group.


### RunThese

A particularly helpful method while developing/enhancing stuff is `RunThese`. The function allows you to run just selected test functions rather than a whole test suite but at the same time process any INI files and execute `Initial` and `Cleanup` if they exist.

`RunThese` offers the following options:

```
      RunThese 1 2    ⍝ Execute test cases 1 & 2 that do not belong to any group.
      RunThese ¯1 ¯2  ⍝ Same but stop just before a test case is actually executed.
      RunThese 'Group1' ⍝ Execute all test cases belonging to group 1.
      RunThese 'Group1' (2 3) ⍝ Execute test cases 2 & 3 of group 1.
      RunThese 'Group1' 2 3   ⍝ Same as before.
      RunThese 'L'    ⍝ Execute all test cases of the group that start with "L" - if there is just one
      RunThese 'L*'   ⍝ Execute all test cases of all groups starting with "L"
      RunThese 'Test_Group1_001'  ⍝ Executes just this test case.
      RunThese 0      ⍝ Run all test cases but stop just before the execution.
```


## Best Practices

* Try to keep your test cases simple and test just one thing at a time, for example just one method at a time. If a method is complex you will be better off creating a group (use the method name as group name) with several simple test cases rather than one complex test case.

* For more complex method create a group with the method name as group name.

* Create everything you need on the fly and tidy up afterwards. Or more precisely, tidy up (left overs!), prepare, test, tidy up again.

* Avoid a test case relying on changes made by an earlier test case. It's a tempting thing to do but you will almost certainly regret this at one stage or another.

* Notice that the DRY principle (don't repeat yourself) can and should be ignored in test cases: any test case should read from top to bottom like an independent story.


## Tester's methods

Note that for convenience there are a couple of Helpers which are shortcuts for some of the methods listed here - see there.


### GetAllTestFns

```
r←GetAllTestFns refToTestNamespace
```
Returns the names of all test functions found in namespace "refToTestNamespace".


### ListTestCases

```
r←ListTestCases refToTestNamespace;list
```

Returns the name and the comment expected in line 1 of all test cases found in "refToTestNamespace".


###  Run

```
{log}←Run refToTestNamespace;flags
```

Runs all test cases in "refToTestNamespace" with error trapping. Broken as well as failing tests are reported in the session as such but they don't stop the program from carrying on.

Sets a global variable `TestCasesExecutedAt` in the namespace hosting the test cases.


### RunBatchTests

```
{(rc log)}←RunBatchTests refToTestNamespace
```

Runs all test cases in "refToTestNamespace" but tells the test functions that this is a batch run meaning that test cases which are reporting to the session for any reason and those in need for a human being for interaction should quit silently. 
Returns 0 for okay or a 1 in case one or more test cases are broken or failed.
This method can run in a runtime as well as in an automated test environment.

Sets a global variable `TestCasesExecutedAt` in the namespace hosting the test cases.


### RunDebug

```
{log}←{stop}RunDebug refToTestNamespace
```

Runs all test cases in "refToTestNamespace" '''without''' error trapping.  If a test case encounters an invalid result it stops. Use this function to investigate the details after "Run" detected a problem.

This will work only if you use a particular strategy when checking results in a test case.

Sets a global variable `TestCasesExecutedAt` in the namespace hosting the test cases.


### RunTheseIn

```
      {log}←testCaseNos RunTheseIn refToTestNamespace
      {log}←(groupName testCaseNos) RunTheseIn refToTestNamespace
```

Like `RunDebug` but it runs just "testCaseNos" in "refToTestNamespace". Use this in order to debug one or some particular test cases.

If a "groupname" was specified then "testCaseNos" might be empty. Then all test cases of the given group are executed.

Sets a global variable `TestCasesExecutedAt` in the namespace hosting the test cases.


### Version

Returns version (x.y.z) and date (yyyy-mm-dd) of the class `#.Tester`


## Examples

The example requires the `FilesAndDirs` workspace which can be downloaded from GitHub

```
      )cs #._FilesAndDirs.TestCases
      #.Tester.EstablishHelpersIn ⎕THIS
      ListHelpers ''
 Run                       ⍝ Run all test cases                   
 RunDebug                  ⍝ Run all test cases with DEBUG flag on
 RunThese                  ⍝ Run just the specified tests.        
 RunBatchTests             ⍝ Run all batch tests                  
 RunBatchTestsInDebugMode  ⍝ Run all batch tests in debug mode (no error trapping) and with stopFlag←1.    
 E                         ⍝ Get all functions into the editor starting their names with "Test_".          
 L                         ⍝ Prints a list with all test cases and the first comment line to the session.  
 G                         ⍝ Prints all groups to the session.                                             
 FailsIf                   ⍝ Usage : →FailsIf x, where x is a boolean scalar                               
 PassesIf                  ⍝ Usage : →PassesIf x, where x is a boolean scalar                              
 GoToTidyUp                ⍝ Returns either an empty vector or "Label" which defaults to ∆TidyUp           
 RenameTestFnsTo           ⍝ Renames a test function and tells acre.                                       
 ListHelpers               ⍝ Lists all helpers available from the `Tester` class.                          
 ∆OK                       ⍝ Constant; used as result of a test function                                   
 ∆Failed                   ⍝ Constant; used as result of a test function                                   
 ∆NoBatchTest              ⍝ Constant; used as result of a test function                                   
 ∆Inactive                 ⍝ Constant; used as result of a test function                                   
 ∆NoAcreTests              ⍝ Constant; used as result of a test function                                   
 ∆WindowsOnly              ⍝ Constant; used as result of a test function                                   
 ∆LinuxOnly                ⍝ Constant; used as result of a test function                                   
 ∆MacOnly                  ⍝ Constant; used as result of a test function                                   
 ∆LinuxOrMacOnly           ⍝ Constant; used as result of a test function                                   
 ∆LinuxOrWindowsOnly       ⍝ Constant; used as result of a test function                                   
 ∆MacOrWindowsOnly         ⍝ Constant; used as result of a test function                                   
 FindSpecialString         ⍝ Use this to search for stuff like "CHECK" or "TODO" enclosed between `⍝` (⍵).
 ∆NotApplicable     
      L
 Test_Copy_001        Exercise `CopyTo`: copy a file to a folder.                                          
 Test_Copy_002        Exercise `CopyTo`: copy a file to a folder with a new name                           
 Test_Copy_003        Exercise `CopyTo`: Try to copy a file that does not exist (but the parent directory does!)
 Test_Copy_004        Exercise `CopyTo`: Try to copy a file that does not exist, not even the parent directory
...
      Run
--- Test framework "Tester" version 3.7.0 from 2018-01-23 -------------------------------------
Searching for INI file Testcases.ini
  ...not found
Searching for INI file testcases_APLTEAM2.ini
  ...not found
Looking for a function "Initial"...
  "Initial" found and successfully executed
--- Tests started at 2018-01-23 07:51:37 on #._FilesAndDirs.TestCases -------------------------
  Test_Copy_001 (1 of 100) : Exercise `CopyTo`: copy a file to a folder.
  Test_Copy_002 (2 of 100) : Exercise `CopyTo`: copy a file to a folder with a new name
...
 ----------------------------------------------------------------------------------------------
   100 test cases executed
   0 test cases failed
   0 test cases broken
Time of execution recorded on variable #._FilesAndDirs.TestCases.TestCasesExecutedAt in: 2018-01-23 07:52:12
Looking for a function "Cleanup"...
  Function "Cleanup" found and executed.
*** Tests done

⍝ We won't show the "Searching for..." bits from now on in order to save space.

      ⊃RunBatchTests
0

      RunThese 2 10     ⍝ Just test cases 2 and 10
--- Test framework "Tester" version 3.7.0 from 2018-01-23 -------------------------------------
   0 test cases executed
   0 test cases failed
   0 test cases broken
*** Tests done

⍝ All test cases belong to groups, so no hits without specifying a group

      G
Test_Copy      
Test_CopyTree  
Test_DeleteFile
Test_Dir       
Test_Misc      
Test_MkDir     
Test_Move      
Test_MoveTree  
Test_RmDir     
Test_ZZZ       
      L'ZZZ'
 Test_ZZZ_997         Checks whether the result of `Version` fits to README.md (required by .Git).   
 Test_ZZZ_998         Checks on two text vectors: "⍝TODO⍝" and "⍝CHECK⍝"; never fails, just reports. 
 Test_ZZZ_999         Check the "Version" function and publish.config and history.txt                

      RunThese 'ZZZ' (999 997)
--- Test framework "Tester" version 3.7.0 from 2018-01-23 -------------------------------------
--- Tests started at 2018-01-23 07:58:49 on #._FilesAndDirs.TestCases -------------------------
  Test_ZZZ_997 (1 of 2) : Checks whether the result of `Version` fits to README.md (required by .Git).
  Test_ZZZ_999 (2 of 2) : Check the "Version" function and publish.config and history.txt
   2 test cases executed
   0 test cases failed
   0 test cases brokenTime of execution recorded on variable ,..
*** Tests done

      RunThese 'ZZZ' ¯997     ⍝ A negative number...
--- Tests started at 2013-01-08 14:35:28  on #.TestCases --------------------------------------------------------
--- Tests started at 2018-01-23 07:59:56 on #._FilesAndDirs.TestCases -------------------------------------------

ExecuteTestFunction[6]             ⍝ ... stop just before the test cases gets executed
      →⎕LC ⍝
 Test_ZZZ_997 (1 of 1) : Checks whether the result of `Version` fits to README.md (required by .Git).
 -----------------------------------------------------------------------------------------------------------------
   1 test case executed
   0 test cases failed
   0 test cases broken

⍝ Use `RunThese 0` for executing all test cases with stops.

RunDebug 1
--- Test framework "Tester" version 3.7.0 from 2018-01-23 --------------------------------------------------------
--- Tests started at 2018-01-23 08:02:08 on #._FilesAndDirs.TestCases --------------------------------------------

ExecuteTestFunction[6]

⍝ Now you can trace into the test case.

      RunDebug 'ZZZ' ¯997   ⍝ Stop at test case number 997 of the group `ZZZ`
--- Tests started at 2011-10-18 07:16:03  on #.TestCases -------------------------------------

ExecuteTestFunction[6]
⍝ Now you can trace into the test case.

⍝ In case test cases break, fail or are not executed for some reason the report contains more details:

--- Test framework "Tester" version 3.7.0 from 2018-01-23
-----------------------------------------------------------------------------------------------------------------
Could not find the class "IniFiles" in #.AcreDesktop_Testcases;
therefore no INI file was processed
Looking for a function "Initial"...
  "Initial" found and successfully executed
--- Tests started at 2018-01-22 19:32:50 on #.AcreDesktop_Testcases
-------------------------------------------------------------------------------------------------------
  Test_AAA_001 (1 of 13) : Issue an (any) acre command and then check whether it's the right version.
- Test_AcreConfig_002 (3 of 13) : Create a project and check weather `acreconfig.txt[CaseCode]` is honored
- Test_AcreConfig_003 (4 of 13) : Create a project and check weather `acreconfig.txt[Load]` is honored.
- Test_AcreConfig_004 (5 of 13) : Create a project and check weather `acreconfig.txt[Open]` is honored.
- Test_AcreConfig_005 (6 of 13) : Create a project and check weather `acreconfig.txt[ProjectSpace]` is honored.
  Test_CreateProject_001 (7 of 13) : Create project check that it is open and number of files in `APLSource/`
  Test_CreateProject_002 (8 of 13) : Create project, check that it is open and the number of files in `APLSource/`
* Test_CreateProject_003 (9 of 13) : Create a project which has just a namespace script.
* Test_CreateProject_004 (10 of 13) : Create project with nested namespace. 
* Test_Run_001 (11 of 13) : Exercise the `run` function on "nameclass".
- Test_Run_002 (12 of 13) : Exercise the `run` function on both "putfile" and "changefile".
  Test_ZZZ_998 (13 of 13) : Checks whether the result of acre's `getVersion` fits "releaseNotes.txt"
 ----------------------------------------------------------------------------------------------------------------
   13 test cases executed
   3 test cases failed (flagged with leading "*")
   0 test cases broken (flagged with leading "#")
   5 test cases not executed because they were inactive (flagged with leading "-")
Time of execution recorded on variable
#.AcreDesktop_Testcases.TestCasesExecutedAt in: 2018-01-22 19:32:51
Looking for a function "Cleanup"...
  Function "Cleanup" found and executed.
*** Tests done

```