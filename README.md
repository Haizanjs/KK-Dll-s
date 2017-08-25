# KK-Dll-s
http://killzonekid.com/arma-64-bit-extensions/ - 

KillzoneKid

Statement:
So, after almost 2 years of working for Bohemia as external contractor, I decided it was time for me to call it a day. Of course, you would like to know why. Well, I cannot really tell you all the details, but I can tell you 2 things: 1. This is not something I decided over a weekend, 2. I gradually lost my “magic” powers to improve the game in any meaningful way. Looking back at things that made into the game because of my involvement directly and indirectly, this was not a bad run, can’t really complain. So this is what is going to happen now. I am quitting Arma as well, as there is no point for me to continue with it. I will leave this website, probably until hosting runs out, but won’t be posting any more Arma related stuff. Same goes for the youtube channel and twitter. Please do not ask me to help you with Arma stuff, as I am going to uninstall the game and will not be able to help you even if I wanted to.

I would also like to say thank you to everyone who supported my efforts in making it a better game and to those who supported me personally and this blog. This is why with my last post I would like to share something I’ve been working on recently. Dwarden was really excited about this, so I cannot just let him down after all. The idea was to be able to seamlessly transfer any client from one server to another, so that the very least one can do is to travel from one map to another without the need to restart the server or the mission. There is an engine solution to this, obviously, which is not publicly available. But it is still possible to “hack” the existing UI functionality, to make it good enough for something to start with.

The following code is for the function, which should be called via remote exec from server. The server has access to the database so it should be possible to sync the transfer of a client between the servers via the database. It will need some work before the framework is developed, if successful this could have huge impact on how Arma games are played. So here is fn_redirectClientToServer.sqf:


#include "\A3\Ui_f\hpp\defineResincl.inc"

if (
    isServer 
    || 
    !(
        isRemoteExecuted 
        && 
        remoteExecutedOwner isEqualTo 2
    )
) 
exitWith {diag_log "Must be remote executed from dedicated server"};

params 
[
    ["_IP", "127.0.0.1"], 
    ["_PORT", "2302"],
    ["_PASS", ""], 
    ["_TIMEOUT", 30]
];

_this resize 0;

onEachFrame format [
"
    private _allDisplays = allDisplays select 
    [
        allDisplays find findDisplay %5, 
        count allDisplays
    ];
    reverse _allDisplays;
    {_x closeDisplay %4} forEach _allDisplays;
    
    onEachFrame 
    {
        findDisplay %6 closeDisplay  %4;
        findDisplay %7 closeDisplay  %4;
        
        onEachFrame
        {
            ctrlActivate (findDisplay %8 displayCtrl %9);
            
            onEachFrame
            {
                private _ctrlServerAddress = findDisplay %10 displayCtrl 2300;
                _ctrlServerAddress controlsGroupCtrl %11 ctrlSetText ""%1""; 
                _ctrlServerAddress controlsGroupCtrl %12 ctrlSetText ""%2"";
                ctrlActivate (_ctrlServerAddress controlsGroupCtrl %14);
                
                onEachFrame 
                {   
                    findDisplay %8 displayCtrl %13 lbData 0 call 
                    {
                        if (diag_tickTime > %18) then
                        {
                            diag_log ""RCTS Timeout (no server)"";
                            onEachFrame {};
                        };  
                    
                        if (_this isEqualTo ""%1:%2"") then
                        {
                            findDisplay %8 displayCtrl %13 lbSetCurSel 0;
                            
                            onEachFrame 
                            {
                                ctrlActivate (findDisplay %8 displayCtrl %15);
                                
                                onEachFrame 
                                {                       
                                    if (diag_tickTime > %18) then
                                    {
                                        diag_log ""RCTS Timeout (cannot join)"";
                                        onEachFrame {};
                                    };
                                    
                                    if (!isNull findDisplay %16) then
                                    {
                                        private _ctrlPassword = findDisplay %16 displayCtrl %17;
                                        _ctrlPassword ctrlSetTextColor [0,0,0,0];
                                        _ctrlPassword ctrlSetText ""%3"";
                                        ctrlActivate (findDisplay %16 displayCtrl %14);
                                    };
                                    
                                    if (getClientStateNumber >= 3) then
                                    {
                                        diag_log ""RCTS Success"";
                                        onEachFrame {};
                                    };
                                };
                            };
                        };
                    };
                };
            };
        };
    };
", _IP, _PORT, _PASS, IDC_CANCEL, IDD_MISSION, IDD_DEBRIEFING, IDD_MP_SETUP, IDD_MULTIPLAYER, 
IDC_MULTI_TAB_DIRECT_CONNECT, IDD_IP_ADDRESS, IDC_IP_ADDRESS, IDC_IP_PORT, IDC_MULTI_SESSIONS, 
IDC_OK, IDC_MULTI_JOIN, IDD_PASSWORD, IDC_PASSWORD, diag_tickTime + _TIMEOUT];


I have tested this on 2 local dedis and was able to transfer from one server to another and back. It even supports passworded servers. One thing that lets it down is the fact that you can see all the loading and UI screens flashing like on LSD trip. So if you want smooth transfer it has to be modded, there is no other way. Here what it looks like from my earlier experiments going from Altis to Stratis and hiding this with a mod:


A few words about technical side of things. This is possible because of using onEachFrame from within onEachFrame so that you run certain portion of code per 1 frame. onEachFrame also survives exiting procedure from the game but gets reset as soon as new mission is received. This is enough to handle server switching. I added 2 timeouts, one in case something is wrong with the server you are trying to connect and second if joining somehow failed (wrong password for example). You can also set duration of timeout, default is 30 seconds. Example of calling this function from the server:

["127.0.0.1", "2312", "abc"] remoteExecCall ["KK_fnc_redirectClientToServer", 7];
This will instruct a client with owner id 7 to transfer from current server to 127.0.0.1:2312, and if it is passworded, use password “abc”. Feel free to modify or improve on it. If you make something cool out of it, I am sure Bohemia might consider giving you an engine solution.

That’s it folks,

KK

P.S. Was asked what displays I modified in one way or another to hide transition. Here they are:

RscDisplayClient, RscDisplayNotFreeze, RscDisplayLoadMission, RscDisplayMultiplayerSetup, RscDisplayMultiplayer, RscDisplayIPAddress, RscDisplayPassword, RscChatListDefault, RscChatListBriefing
