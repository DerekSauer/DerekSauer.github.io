## Addon Independent Mission Editing

### Abstract
It is possible in Arma to create missions that are not strongly dependent on addons if you're willing to do some extra work. This primarily involves selecting items and units that are present in addons and falling back to a different addon if the first one is not present, or use stock units and items. It requires you to detect the presence of an addon and utilize simple if/else logic to select the most appropriate classname.

### Limits and complications
This method works best for items, weapons, uniforms and gameplay modifications like Task Force Radio or ACE3. Replacing whole units such as soldiers or vehicles is also rather easy *if* those units are spawned by a script during the mission. Replacing units placed in the editor is much more difficult. It is possible to do but involves copying the original unit's waypoints, combat mode, ROE, speed, etc... and applying them to the new unit. You will also need to develop a way to execute the original unit's init code if there was any and make sure the new unit is given the same variable name (the name field in the unit's editor attributes) if you have any other scripts that reference that variable. 

It will be necessary to avoid allowing addons to pollute your mission file with their dependencies. If Arma tries to load a mission but the addon(s) listed in that mission's dependency list are not present, the mission will not load. Simply never using an addon's units or features is one way to do this but it very easy to add an addon item or module by mistake. It is simple to prune addon dependencies from the mission file itself (mission.sqm) since they are just lines of text at the top of the mission file in an easy to understand format. Another way to avoid unnecessary dependencies is to simply edit your mission with no optional addons loaded. Addons won't get added to the mission's dependency list if they weren't present when the mission was saved. If your mission absolutely requires an addon to function (terrains are a good example) then having it loaded is fine. 

You will need to write some code. This can't be avoided if you want addon independence. You will need to be comfortable navigating Arma's config database (configfile) using the config viewer. The config viewer can be accessed in the editor under *tools* in the menu. You can also access it in a mission with the debug console.  

### Change your way of doing things
Like I wrote above; addon independence doesn't work very well if you're placing addon content in the mission editor. Assume that placing an addon unit (E.g.: RHS Russian bad guys) or items (E.g.: ACE medical crates) in the editor will make that addon a hard dependency unless you're willing to do the work to programatically replace those units or items on mission load with something else if the addon(s) you used are not present. Some addons are probably pretty safe to use; I don't see us getting rid of RHS or ACE in the near future, but TRYK would be a good example of an addon that has come and gone and broken things each way. Medical supplies that work without or without ACE are easy to do as you'll see below. Equipping your unit's with addon gear and falling back to something else is also very easy. Replacing whole units that have been pre-placed in the editor is hard. Doesn't mean you shouldn't do it, just be aware that you might have to fix that mission in the future if something you used disappears. 

When I edit a mission I usually load only the addons I plan to have as hard dependencies (CUP Terrain for example). Anything else is added programatically in the manner seen below. That's just my preferred way to do things since its very safe and easy to do for someone comfortable with SQF. It can be a little tedious to do it this way since you should still test your code and that means having to exit the game and re-launch it with the addons you're optionally using to make sure you didn't make a mistake in your code. Then re-launch the game again without the optional addons loaded. 

### Detecting an addon... And what to do about it
Arma provides a command called [isClass](https://community.bistudio.com/wiki/isClass) which allows you to query the config database for the presence of classes (a bunch of text that describes something, there's dozens of types of classes). Nearly everything in Arma is represented by a class but the ones we're most interested in are classnames (the unique identifier used to reference the type of every item or unit we can place in a mission) and patches. We're not talking about the patches you put on your shoulder. A patch is the term Arma internally uses for addons. To detect an addon you query if its patch name exists with [isClass](https://community.bistudio.com/wiki/isClass). If the addon you want is present, do something; if it isn't, do something else.

An example of [isClass](https://community.bistudio.com/wiki/isClass), this code could be executed in the init field of a storage box. 
```sqf
// Check to see if ACE Medical is loaded
if isClass (configfile >> "CfgPatches" >> "ACE_Medical") then {
    // Ace Medical exists, make Cooked less depressed
    this additemcargoGlobal ["ACE_Morphine", 25];
    this additemcargoGlobal ["ACE_Epinephrine", 25];
    this additemcargoGlobal ["ACE_PackingBandage", 50];
    this additemcargoGlobal ["ACE_ElasticBandage", 50];
    this additemcargoGlobal ["ACE_bloodIV", 10];
    this additemcargoGlobal ["ACE_tourniquet", 25];
    this additemcargoGlobal ["ACE_bodyBag", 100000];
} else {
    // Ace Medical doesn't exist, fallback to boring old FAKs
    this additemcargoGlobal ["FirstAidKit", 20];
};
```

You can lookup relevant patch names with the config viewer. They are located under the cfgPatches category. Some addons, such as ACE3, are modular and have different patch names for each of their components. Its better to look for a specific component rather than the root component of the addon (ACE_Medical instead of ACE_Main in this case) as there is no guarantee the server will have all components loaded (ACE_gForces is a good example of a horrible module no one sane should load). 

You can also look up a specific classname if you're really interested in only that item rather than everything in the addon which might be useful for addons authored by insane spurgs who have a tendency to change classnames if they don't think they're getting enough attention on the internet. 

An example of querying a specific classname. This could be called in a soldier unit's init field.
```sqf
// Strip the unit's weapon and magazines
removeAllWeapons this;

// Check to see if HLC's AK74 is available
if isClass (configfile >> "CfgWeapons" >> "hlc_rifle_ak74") then {
    // It is, give the unit a gun and ammo
    [this, "hlc_rifle_ak74", 6] call BIS_fnc_addWeapon;
} else {
    // Guess HLC changed its classnames again, let's try RHS
    if isClass (configfile >> "CfgWeapons" >> "rhs_weap_ak74m") then {
        // RHS's AK exists it seems
        [this, "rhs_weap_ak74m", 6] call BIS_fnc_addWeapon;
    } else {
        // Everything is broken and horrible, enjoy your stock weapon
        [this, ""arifle_Katiba_F"", 6] call BIS_fnc_addWeapon;
    };
};
```

### Server/Client side settings
Some addons don't actually add any items to the game or work automatically on their own. In many cases you can apply settings for an addon that doesn't exist without it having any effect on your mission. 

ACE is a good example of this. You can configure all of ACE's settings without having to place ACE's modules in the mission. This way ACE does not get added to your mission's dependency list, but you can still control it and if ACE doesn't exist on the server the mission isn't broken. This involves adding ACE settings to your mission's description.ext file. Look up the [ACE Settings Framework](http://ace3mod.com/wiki/framework/settings-framework.html) for instructions. This may even be easier to use for some people since you won't need to go and double click on a bunch of modules you placed in the mission until you find the one you want; just search a text file. 

ASR_AI works automatically if it is loaded on the server and additional configuration can be done server side in initServer.sqf. You can query the presence of the ASR_AI addon by looking up its patch name prior to applying your settings but its harmless if you don't since ASR_AI's settings are nothing more than a set of global variables with names you will probably never mistakenly use in your own code, for example:

```sqf
// The conditional is optional and harmless if you omit it even if ASR_AI isn't loaded
if isClass (configfile >> "CfgWeapons" >> "hlc_rifle_ak74") then {
    // Prevent ASR_AI from overriding my own unit skill settings
    asr_ai3_main_setskills 		= 0;
    publicVariable "asr_ai3_main_setskills";
    asr_ai3_main_joinlast 		= 0;
    publicVariable "asr_ai3_main_joinlast";
    asr_ai3_main_removegimps	= 300;
    publicVariable "asr_ai3_main_removegimps";
    asr_ai3_main_rearm          = 40;
    publicVariable "asr_ai3_main_rearm";
    asr_ai3_main_gunshothearing = 1.1;
    publicVariable "asr_ai3_main_gunshothearing";
    asr_ai3_main_radiorange     = 250;
    publicVariable "asr_ai3_main_radiorange";
};
```

Task Force Radio is much the same; "It Just Works"... Most of the time. TFR will automatically replace the stock radio with one if its own appropriate to the unit's side. If you want to customize radio settings in initServer.sqf you can without needing a patch name check for the same reason you don't need one for ASR_AI. 

### I polluted my mission with dependencies, now what?
Eden is pretty good at cleaning up after itself so if you haven't added addon content to the mission in the mission editor or have deleted it you probably don't need to do this. But just in case; Make sure whatever addons unit or modules you added to the mission are deleted. Save your mission as plain text by turning off *Binarize the Scenario File* when you save your mission. You may need to use the Save As dialog on an existing mission. You can also turn off this feature permanently in Eden's settings if you like. Use your favorite text editor to open the mission.sqm file for your mission. A mission file is nothing more than blocks of classes. Look for the *addons* block which is normally near the top of the mission file. Find the offending addon entry and delete the line, including the trailing comma if there is one. 

Example of removing an addon dependency leftover in a mission file. I got rid of the ACE Pointing module in the editor, preferring to use the ACE Setting Framework, but the addon dependency is still there. Delete *ace_finger*.
```sqf
version=52;
class EditorData
{
	moveGridStep=1;
	angleGridStep=0.2617994;
	scaleGridStep=1;
	autoGroupingDist=10;
	toggles=1;
	class ItemIDProvider
	{
		nextID=3;
	};
	class Camera
	{
		pos[]={2141.7129,96.086861,3318.9502};
		dir[]={0.94464785,-0.20769802,-0.25397712};
		up[]={0.20057555,0.97819227,-0.05392658};
		aside[]={-0.25963926,3.426976e-008,-0.96570718};
	};
};
binarizationWanted=0;
addons[]=
{
	"A3_Characters_F_BLUFOR",
	"a3_characters_f",
	"ace_finger"                // I deleted the ACE Pointing module but this is still here; delete this line.
};
```

Now this mission will load even if ACE isn't being used. Arma will just ignore the ACE settings imported through the settings framework if ACE is not loaded since they're just a bunch of global variables in the mission namespace. This wouldn't have happened in the first place if I hadn't loaded ACE to begin with when launching Arma to edit my mission. 

### More Examples
These are take from some of my own missions.

---
Setup some ACE items:
```sqf
// initPlayer.sqf

// Medical stuff
if isClass (configfile >> "CfgPatches" >> "ACE_Medical") then {
    // Basic ACE Medical load out for all players, medics get more stuff in their class specific loadout
    for [{_i = 0}, {_i < 4}, {_i = _i + 1}] do
    {
        player addItemToUniform "ACE_fieldDressing";
    };

    player addItemToUniform "ACE_morphine";
    player addItemToUniform "ACE_epinephrine";
} else {
    for [{_i = 0}, {_i < 2}, {_i = _i + 1}] do
    {
        // Ace Medical isn't being used
        player addItemToUniform "FirstAidKit";
    };
};

// Earplugs
if isClass (configfile >> "CfgPatches" >> "ace_hearing") then {
    player addItemToUniform "ACE_EarPlugs";
};
```
---
Give all the OPFOR creepy gas masks if TAC is loaded:
```sqf
// initServer.sqf
{
    if (side _x == EAST) then {
            if isClass (configfile >> "CfgGlasses" >> "TAC_SF10") then {
                removeGoggles _x;
                _x addGoggles "TAC_SF10";
            };
        };
} foreach allUnits;
```
---
Allow a player to take the easy way out without using the module. Doesn't need a patch name check since its nothing more than a global variable with a funky name you probably won't use yourself:
```sqf
// initPlayer.sqf
murshun_easywayout_canSuicide = true;
```
---
Let the nicotine addicts have their candy if that addon is available. If it ever breaks horribly and we get rid of it, this mission will still work without having to modify it:
```sqf
// fn_suckAsh.sqf
// Turn any container into a humidore.
// Param: _boxName (object) - The container to fill with coffin nails.
// Example: 
//      [humidore01] call Sauer_fnc_suckAsh;

params ["_boxName"];

// Smokes
if isClass (configfile >> "CfgPatches" >> "EWK_CIGS") then {
    _boxName additemcargoGlobal ["EWK_Cigar1", 10];
    _boxName additemcargoGlobal ["EWK_Cig1", 20];
};

// Fire
if isClass (configfile >> "CfgPatches" >> "murshun_cigs") then {
    _boxName additemcargoGlobal ["murshun_cigs_matches", 20];
    _boxName additemcargoGlobal ["murshun_cigs_lighter", 5];
};

// Consolation prize
if (!isClass (configfile >> "CfgPatches" >> "EWK_CIGS") &&  !isClass (configfile >> "CfgPatches" >> "murshun_cigs")) then {
    _boxName additemcargoGlobal ["Chemlight_Red", 5];
    _boxName additemcargoGlobal ["Chemlight_Green", 5];
    _boxName additemcargoGlobal ["Chemlight_Blue", 5];
    _boxName additemcargoGlobal ["Chemlight_Yellow", 5];
};
```
---
Spawn a Russian spec ops group if they're available, otherwise spawn CSAT spec ops group:
```sqf
if isClass (configfile >> "CfgGroups" >> "East" >> "rhs_faction_vdv" >> "rhs_group_rus_vdv_infantry_recon" >> "rhs_group_rus_vdv_infantry_recon_squad") then {
    _group = [getMarkerPos "dudeSpawner", EAST, (configfile >> "CfgGroups" >> "East" >> "rhs_faction_vdv" >> "rhs_group_rus_vdv_infantry_recon" >> "rhs_group_rus_vdv_infantry_recon_squad")] call BIS_fnc_spawnGroup;
} else {
    _group = [getMarkerPos "dudeSpawner", EAST, (configfile >> "CfgGroups" >> "East" >> "OPF_F" >> "Infantry" >> "OIA_ReconSquad")] call BIS_fnc_spawnGroup;
};

// Do more stuff with the new group...
```