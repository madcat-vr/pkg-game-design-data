# Overview

Main purpose of the plugin is to read data from Google spreadsheets into the objects in your Unity project. When invoked in editor, can read into ScriptableObjects and MonoBehaviours (in scenes or in prefabs). When invoked in run-time, can read into any objects consisting of fields or properties. Mapping between fields/properties and spreadsheet cells is done in a declarative way, so that you don't need to write any parsing code. Consider the following data definition and the spreadsheet it reads from:

```csharp
[Serializable]
public class Answer
{
    [Column("Answer")] public string name;
    [Column("Correct")] public bool correct;
}

[Serializable]
public class Question
{
    [Column("Question")] public string name;
    [Block("Answer")] public Answer [] answers;
}

public class QuizData : ScriptableObject
{
    [SerializeField, Spreadsheet] string spreadsheetId = "";

    [SerializeField, Sheet("Questions"), Block("Question")]
    public List<Question> questions;
}
```

```
       | A                                              | B              | C       |
-------|------------------------------------------------|----------------|---------|
  1    | Question                                       | Answer         | Correct |
-------|------------------------------------------------|----------------|---------|
  2    | What is the lightest existing metal?           | Aluminium      | Yes     |
  3    |                                                | Iron           |         |
  4    |                                                | Zinc           |         |
  5    |                                                | Chromium       |         |
  6    | Which planet is nearest the sun?               | Earth          |         |
  7    |                                                | Jupiter        |         |
  8    |                                                | Mercury        | Yes     |
  9    |                                                | Mars           |         |
  10   | What is the most fractured human bone?         | Fibula         |         |
  11   |                                                | Clavicle       | Yes     |
  12   |                                                | Talus          |         |
  13   |                                                | Metacarpal     |         |
  14   | What is the largest number of five digits?     | 99999          | Yes     |
  15   |                                                | 100000         |         |
  16   |                                                | 10000          |         |
  17   |                                                | 55555          |         |
```

## Attributes

Mapping to the spreadsheets is achieved with attributes.

#### ```[SpreadsheetId]```

Indicates a member that contains the spreadsheet id, if Read operations is triggered without spreadsheet id explicitly specified, it will try to get the id from the field or property marked with [SpreadsheetId]. See the "Spreadsheet ID specification options" section below for details.

#### ```[DefaultSheet("sheet_name")]``` and ```[Sheet("sheet_name")]```

**DefaultSheet** is applicable to classes and allows omitting sheet name specification on individual members. **Sheet** is applicable to fields and properties and specifies the sheet to use when reading this data into this member. See how source sheet name for ```settings``` and for ```questions``` are specified in a different way:

```csharp
[Serializable, DefaultSheet("Settings")]
public class Settings
{
    [Row("AnswerTimeout")] public float answerTimeout;
    [Row("WinningStreak")] public int winningStreak;
}

public class QuizData : ScriptableObject
{
    [SerializeField] Settings settings;
    public Settings Settings => settings;

    [SerializeField, Sheet("Questions"), Block("Question")]
    public List<Question> questions;
}
```

#### ```[Column("column_name")]```

You can be flexible on where to look for the "column_name" that acts as a key identifying the column. By default it's looked up in the first row, however reader respects frozen rows and will look for the given key in all frozen rows. If it's found, all column cells past the key area are returned as values.

If this attribute is specified on Array of List it will be populated with all values found in the column, otherwise only first value is used.

#### ```[Row("row_name")]```

Same idea as for columns, but key is looked up in the leftmost cells of a row. By default first cell in a row is treated as key area, if you have some frozen columns, all of them will be considered to be a key area. And when reading cell values for the row, key area ignored.

Both Column and Row attributes have an additional argument ```ignoreEmptyCells```, that is ```true``` by default. Set it to ```false``` if you need empty values present in the list.

#### ```[Cell("column_name", "row_name")]```

Returns a single value on the intersection of a given column and row. Can also be applied to Arrays and Lists, but in this case they will be populated with a single element.

#### ```[Block("delimiter_column")]```

You can group your data to blocks, any non-empty cell in the column identified by ```delimiter_column``` key is considered a block start, any further rows with the same or empty value in the delimiter column will belong to the same block. New value encountered in the delimiter column starts a new block. See how Questions and Answers are defined in the example above. It also illustrates blocks nesting, which can be of unlimited depth.

## Enums

You can bind spreadsheet cells to values of enum types. In this case you specify the names in your cells. For the following code, valid cell values are: **Warrior**, **Mage**, **Rogue**, **Priest**

```csharp
public enum CharacterClass
{
    Warrior,
    Mage,
    Rogue,
    Priest
}

[Column("Class")] public CharacterClass characterClass;
```

Flags are also supported.

```csharp
[Flags]
public enum Inventory
{
    None = 0,
    Sword = 1,
    MagicStaff = 2,
    Armor = 4,
    Potion = 8
}

[Column("Inventory")] public Inventory inventory;
```

Valid cell values for this enum are either comma separated or | separated names:
- Sword | Armor
- Sword
- MagicStaff, Potion

## Loose Header Matching

By default when you specify the key (column or row name), it will be looked up as a case-sensitive exact match. This way if your columns are "CASE", "case" and "Case", you'll be able to address them correctly. Same with rows. In case the key is not found as the exact match, case-insensitive search is attempted. And then if it fails, the search is performed without regard to whitespace.

In the **Feature Reference** example you'll find the following data, while the actual column names are: **HardcoreMode**, **Music Volume**, **Sound Volume**:

```csharp
[Column("hardcoremode")] public bool hardcore { get; set; }
[Column("MusicVolume")] public int musicVolume { get; set; }
[Column("s Ou n D vo Lu mE")] public int soundVolume { get; set; }
```

# Setup Instructions

Here's the summary of steps on how to enable access to your google spreadsheets using Sheets API:

- Create a project on Google Cloud Platform (https://console.cloud.google.com)
- Enable Google Sheets API for this project
- Create Service Account
- Add JSON Key for the Service Account

# Editor Options

First major use-case for this plugin is synchronizing your data objects to spreadsheets in edit mode. You can target either ScriptableObjects or MonoBehaviours in prefabs or in scenes. This way all web requests to fetch the data and all the reflection-based operations to access the target objects are happening in editor and in the run-time you can access the data as if it was entered manually. Has the following pros and cons:
- ✅ Does not require the internet connection in the run-time.
- ✅ Smaller app build size, because no extra code is included in builds.
- ✅ Zero performance loss on loading the data in the run-time, because it's already there when you run.
- ❌ You can't update data after building.

> 🤓 **Tip:** In many cases you can still have the benefit of live data updates at the cost of requiring internet connection and extra build size, but for development builds only. This way you can quickly iterate on the data values while testing internally, and then lock the data and disable run-time updates in the release builds.

## Editor Settings

As mentioned in the setup section, you'll need a credentials file in json format to access the spreadsheets. Location of this file has to be specified in **Project Settings**, in the **Game Design Data section**. There's a field named **Credential Path** and by default it's relative to the project root folder. You may choose to have the credentials file outside of the project.

Another option configurable here is the **Verbose** flag that enables the log output from the plugin.

These settings will apply for spreadsheet reading operations triggered by pressing the **Read** button next to the field with ```[SpreadsheetId]``` attribute. Or for the read operations invoked for ```MadCat.GameDesignData.Editor.SpreadsheetReader```

## Reader Editor API

```csharp
namespace MadCat.GameDesignData.Editor
{
    public class Settings
    {
        public Settings(string credentialPath, bool relative = true, bool verbose = false);
        public static void Save(Settings settings);
        public static Settings Load();
    }

    public class SpreadsheetReader
    {
        public SpreadsheetReader(Settings settings);
        
        public Object Read(Object destination);
        public Object Read(string spreadsheetId, Object destination);
        
        public Object Read(Object destination, SerializedObject serializedDestination);
        public Object Read(string spreadsheetId, Object destination, SerializedObject serializedDestination);
    }
}
```

Editor version of ```SpreadsheetReader``` can only read to destinations that are ```UnityEngine.Objects```, as it relies on ```SerializedObjects``` and ```SerializedProperties``` to modify the destination object. It also means that if you have a property or field that's not public and doesn't have ```[SerializeField]```, then in this case even if it's tagged with data source attribute, it will be ignored.

Note that if you're not supplying the spreadsheetId to the ```Read``` method, it has to be in the object itself, see section **Spreadsheet ID specification options** further below.

# Runtime Options

Optionally, it is possible to sync with the spreadsheet in run-time too. In order to do so, add ```MC_GDD_ENABLE_RUNTIME_SYNC``` to **Project Settings**. This will enable **MadCat.GameDesignData.Runtime** assembly. By default, this assembly is not built and therefore dependent dlls are not included in builds as well. 

You don't need to remove/#if any calls to ```MadCat.GameDesignData.SpreadsheetReader``` in your code, when runtime sync is disabled, there will still be ```MadCat.GameDesignData.SpreadsheetReader``` implementation with the same public interface, but doesn't read anything.

## More Destination Options

Runtime spreadsheet reader relies on reflection to write to the destination object. It means that it's not limited to ```UnityEngine.Object``` as the destination type. Destination object doesn't even have to be ```[Serializable]```.
Additionally, runtime reader can write to the properties, unlike from editor version that only writes to fields.

## Reader Runtime API

```csharp
namespace MadCat.GameDesignData
{
    public class SpreadsheetReader
    {
        public SpreadsheetReader(TextAsset credentialAsset, bool verbose = false);
        public SpreadsheetReader(string credentialJson, bool verbose = false);

        public T Read<T>() where T : new();
        public T Read<T>(string spreadsheetId) where T : new();

        public T Read<T>(T destination);
        public T Read<T>(string spreadsheetId, T destination);

        public object Read(object destination);
        public object Read(string spreadsheetId, object destination);
    }
}
```

Note that there's no separate Settings class for runtime version of the reader. All you need to initialize the reader is provided directly in constructor. Here you can also choose how to locate your credentials, it can be the asset in the project that you reference explicitly or get from Resource, etc. Or if you know how to obtain the credential json from some external source in yor app specific way, then constructor that takes credential as ```string``` is your choice.

# Spreadsheet ID specification options

Spreadsheets are identified by their ID, that is a part of spreadsheet URL. For example in this URL
https://docs.google.com/spreadsheets/d/2dlUKzpU7KyXUQQKihI_mQFgYfkNZPr1WMTb2xhc-97M/edit
ID would be: 2dlUKzpU7KyXUQQKihI_mQFgYfkNZPr1WMTb2xhc-97M

You have several options where to specify the ID depending on your use case:
- As default value in Spreadsheet attribute: ```[Spreadsheet("2dlU...c-97M")] string id;```
- As getter with Spreadsheet attribute: ```[Spreadsheet] string id => "2dlU...c-97M";```
- In the inspector for serialized field: ```[SerializeField, Spreadsheet] string id;```
- As an explicit argument to the Read call: ```new SpreadsheetReader(credentials).Read("2dlU...c-97M", destination);```

All options have their pros and cons. For instance first two options with the hardcoded IDs are not applicable if you don't want your document IDs to be visible in code, saved to source control, etc. That also doesn't work if you want different IDs per instance of an object. Last option on the other hand is less convenient, but allows greater flexibility. For example in runtime version of SpreadsheetReader you can read from the document whose ID is entered dynamically in the app UI, which is not achievable with other options. And in the editor version of SpreadsheetReader you can potentially use different versions of data depending on your build settings if you're adding data synchronization as a part of your CI/CD pipeline.

# Dependencies Handling

Plugin comes with several dlls in the Dependencies folder. They are included for all build targets by default. In case you have some of them already present in your project or bundled with other plugins, you have an option to exclude them by using the following symbols:
- ```MC_GDD_EXCLUDE_DEPENDENCIES``` - Excludes all dependencies DLLs contained in this plugin.
- ```MC_GDD_EXCLUDE_GOOGLE_APIS``` - Excludes Google.Apis.dll contained in this plugin.
- ```MC_GDD_EXCLUDE_GOOGLE_APIS_AUTH``` - Excludes Google.Apis.Auth.dll contained in this plugin.
- ```MC_GDD_EXCLUDE_GOOGLE_APIS_CORE``` - Excludes Google.Apis.Core.dll contained in this plugin.
- ```MC_GDD_EXCLUDE_GOOGLE_APIS_SHEETS_V4``` - Excludes Google.Apis.Sheets.v4.dll contained in this plugin.
- ```MC_GDD_EXCLUDE_NEWTONSOFT_JSON``` - Excludes Newtonsoft.Json.dll contained in this plugin.
- ```MC_GDD_EXCLUDE_SYSTEM_CODEDOM``` - Excludes System.CodeDom.dll contained in this plugin.
- ```MC_GDD_EXCLUDE_SYSTEM_MANAGEMENT``` - Excludes System.Management.dll contained in this plugin.

# Examples

Plugin comes with two examples illustrating major use-cases.

**QuizGame** - focuses on edit-time data synchronization. Game itself is a sequence of questions presented to player, with four answers each. And the whole set of questions, answers and some settings are stored in the scriptable object. It can be edited manually, but the idea is that it's fetched from the spreadsheet explicitly by pressing the **Read** button next to the spreadsheet id field in the scriptable object.

Once data is fetched from the spreadsheet, you can build the game and it will have the data without connecting anywhere to get it.

**Feature Reference** - is a bit more artificial, it doesn't have any game logic, but instead shows all existing ways of binding spreadsheet data to the object in the game. And in particular features the option of synchronizing in run-time. To enable run-time synchronization, don't forget to add ```MC_GDD_ENABLE_RUNTIME_SYNC``` to **Project Settings**. Then once you play the **FeatureReference** scene it will synchronize two objects, one is mono behavior consisting of public fields with all kinds of data binding attributes on them. Another object is a plain System.Object with same data represented as properties. You can review the property values in the inspector using the Debug view.

Run-time synchronization illustrates synchronizing to the spreadsheet whose id is provided to the Read method as an argument.

Finally, there's a prefab **FieldBasedPrefab** with the same data, but this time with spreadsheet id specified in prefab inspector, so that **Read** button is available in inspector to synchronize in edit-time.

