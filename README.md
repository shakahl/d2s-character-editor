I pay attention to a project `d2es-stuff/tools/esCharView/CharacterEditor` in the repository forked from [nooperation/d2es-stuff](https://github.com/nooperation/d2es-stuff). 

**CharacterEditor** is .NET library for editing Diablo 2 character files (`.d2s`).

Several changes from me:
* fix a bug in inventory
* add feature to get bytes of `d2s` to save data to a file
* feature to add new items to a character
* add resources for 1.13c to make it possible edit original d2s files.

This library is used in online service that can edit characters through REST API, there is PHP interface to consume it https://github.com/pvpgn/d2smanager-api




## How to use CharacterEditor.dll

Put `Resources` directory in the root with your application, then you can pass different "mods" (a directory name) to `SaveReader` constructor. For 1.13c you should use `new SaveReader("1.13c")`.

It is possible to set a directory with Resources in code:
```c#
CharacterEditor.Resources.CurrentDirectory = @"C:\MyProject\App_Data\";
```

Read character into object
```c#
var reader = new SaveReader("1.13c")

var charBytes = File.ReadAllBytes("harpywar.d2s");
var d2s = reader.Read(charBytes);
``` 


Split all character items into different files
```c#
var items = new List<Item>();
items.AddRange(d2s.Inventory.PlayerItems);
items.AddRange(d2s.Inventory.CorpseItems);
items.AddRange(d2s.Inventory.GolemItems);
items.AddRange(d2s.Inventory.MercItems);

for (var i = 0; i < items.Length; i++)
{
	File.WriteAllBytes(i + ".d2i", items[i].GetItemBytes());
}
```

Add new item from `.d2i` file to a character's inventory 
```c#
var itemBytes = File.ReadAllBytes("item.d2i");
var item = new Item(itemBytes, true)

// add to character inventory
d2s.Inventory.PlayerItems.Add( FixItem(item) );

// set default item position in inventory in left-top corner
public static Item FixItem(Item item)
{
	var newItem = new Item(item.GetItemBytes(), true);

	newItem.Id = 0;
	newItem.Location = (uint)Item.ItemLocation.Stored;
	newItem.PositionOnBody = (uint)Item.EquipmentLocation.None;
	newItem.PositionX = 0;
	newItem.PositionY = 0;
	newItem.StorageId = (uint) Item.StorageType.Inventory;

	return newItem;
}
```

Remove item from a character 
```c#
// item you want to remove
Item findItem;

// find item on body
if ( findInItems(d2s.Inventory.PlayerItems, findItem) )
{
	d2s.Inventory.PlayerItems.Remove(item);
}
// find in merc
if ( findInItems(d2s.Inventory.MercItems, findItem) )
{
	d2s.Inventory.MercItems.Remove(item);
}
// find in golem
if ( findInItems(d2s.Inventory.GolemItems, findItem) )
{
	d2s.Inventory.GolemItems.Remove(item);
}
// find in corpse
if ( findInItems(d2s.Inventory.CorpseItems, findItem) )
{
	d2s.Inventory.CorpseItems.Remove(item);
}

// find an item in items list
private findInItems(List<Item> items, Item findItem)
{
	foreach (var item in items)
	{
		if (ByteArrayCompare(item.GetItemBytes(), findItem.GetItemBytes()))
		{
			return true;
		}
	}
	return false;
}
// just compare two byte arrays
static bool ByteArrayCompare(byte[] a1, byte[] a2) 
{
    return StructuralComparisons.StructuralEqualityComparer.Equals(a1, a2);
}
```

Save modified character
```c#
File.WriteAllBytes("newchar.d2s", d2s.GetBytes());
```



Use IClonable interface to work with many characters without preloading resources each time:
```c#

foreach (var file in Directory.GetFiles(@"C:\pvpgn\var\charsave"))
{
	ReadCharacter(file);
}

private static void ReadCharacter(string fileName)
{
	var charBytes = File.ReadAllBytes(fileName);
	var d2s = CharManager.Read(charBytes); 
	
	// other manipulates with "d2s" object
	...
}

public static class CharManager
{
	private static SaveReader _reader;
	public static SaveReader Reader
	{
		get
		{
			// lazy loading resources
			if (_reader == null)
			{
				// set working directory
				CharacterEditor.Resources.CurrentDirectory = Directory.GetCurrentDirectory(); 
				// load "ItemDefs" from Resources
				_reader = new SaveReader("1.13c");
			}
			return (SaveReader)_reader.Clone(); // create new instance
		}
	}
	
	public static SaveReader Read(byte[] charBytes)
	{
		return Reader.Read(charBytes);
	}
}
``` 

