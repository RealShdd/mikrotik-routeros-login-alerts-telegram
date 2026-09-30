# mikrotik-routeros-login-alerts-telegram
A lightweight RouterOS script that monitors router login events and sends real-time alerts to Telegram — no external dependencies, written entirely routeros script
<img width="337" height="104" alt="image" src="https://github.com/user-attachments/assets/7e6ebb17-9dc2-4152-9a9a-58aaf8358d19" />
<img width="401" height="119" alt="image" src="https://github.com/user-attachments/assets/06c5c3bd-bf18-4cc0-9bf7-cb537721dcc1" />


## Features
- ✅ Alerts on successful logins (toggleable)
- ⚠️ Alerts on failed login attempts (always on)
- Batches up to 5 events per cycle to avoid Telegram rate limits
- Tracks state across reboots using global variables
- Self-healing if the log buffer clears or resets

## Requirements
- RouterOS v7.x
- A Telegram bot token and chat ID

```rsc
/system/script remove "TLG-LoginAlert";
/system/scheduler remove "TLG-LoginAlert-Sch";

/system/script add name="TLG-LoginAlert" source={
    # === toggle mode ===
    # 1 = turn on successful logins alerts
    # 0 = disable successful logins alerts (failed login alerts only)
    :local okLoginAlert 1;
    # bot token and ChatID
    :local BotTkn "<REPLACE-WITH-UR-BOT-TOKEN>";
    :local ChatID "<REPLACE-WITH-UR-ID>";
    ##########################
    ##########################
    # Don't change below
    :global LginCnt;
    :global FailCnt;
    :local devId [/system identity get name];
    :local devBRD [/system/resource/get board-name ];
    :local maxcnt 5;
    :if ([:typeof $LginCnt] = "nothing") do={ :set LginCnt 0 };
    :if ([:typeof $FailCnt] = "nothing") do={ :set FailCnt 0 };
    :local TlgUrl ("https://api.telegram.org/bot" . $BotTkn . "/sendMessage");
    ### OK ALERT ###
    :local OkLogs [/log print as-value where topics~"acc" and topics~"info" and message~"logged in"];
    :local OkCnt [:len $OkLogs];
    :if ($OkCnt > $LginCnt) do={
        :local startIdx $LginCnt;
        :if (($OkCnt - $LginCnt) > $maxcnt) do={
            :set startIdx ($OkCnt - $maxcnt);
        }
        :if ($okLoginAlert = 1) do={
            :for i from=$startIdx to=($OkCnt - 1) do={
                :local entry ($OkLogs->$i);
                :local eTime ($entry->"time");
                :local eMsg ($entry->"message");
                :local msgText ("\E2\9C\85OK Login\n\E2\8C\A8 : " . $devId . " / $devBRD\n\F0\9F\93\9D" . $eMsg . "\n\E2\8C\9A" . $eTime);
                :local postData ("chat_id=" . $ChatID . "&text=" . $msgText);
                :do {/tool fetch url=($TlgUrl) http-method=post http-header-field="Content-Type: application/x-www-form-urlencoded" http-data=($postData) check-certificate=no output=none;
                } on-error={/log error "TG-CheckLogin: sending login OK notification failed";}
            }
        }
        :set LginCnt $OkCnt;
    }

    ### FAIL ALERT ###
    :local FailLogs [/log print as-value where topics~"error" and topics~"crit" and message~"login failure"];
    :local failcnt [:len $FailLogs];
    :if ($failcnt > $FailCnt) do={
        :local startIdx $FailCnt;
        :if (($failcnt - $FailCnt) > $maxcnt) do={
            :set startIdx ($failcnt - $maxcnt);}
        :for i from=$startIdx to=($failcnt - 1) do={
            :local entry ($FailLogs->$i);
            :local eTime ($entry->"time");
            :local eMsg ($entry->"message");
            :local msgText ("\E2\9A\A0 WARNING LOGIN FAILED\n\E2\8C\A8 : " . $devId . " / $devBRD\n\F0\9F\93\9D" . $eMsg . "\n\E2\8C\9A" . $eTime);
            :local postData ("chat_id=" . $ChatID . "&text=" . $msgText);
            :do {/tool fetch url=($TlgUrl) http-method=post http-header-field="Content-Type: application/x-www-form-urlencoded" http-data=($postData) check-certificate=no output=none;
            } on-error={/log error "TG-CheckLogin: sending login notification FAIL failed";}
        }
        :set FailCnt $failcnt;
    }
}

/system/scheduler add name="TLG-LoginAlert-Sch" interval=15s on-event="/system script run TLG-LoginAlert" comment="Alerts successful and/or failed logins on Telegram";

```
