# CAAVPlugin (Minecraft 1.12.2)

## Build
Requirements: Java 8 and Maven.
From this folder run:
    mvn package
The jar will be `target/CAAVPlugin-1.0.0.jar`.
Copy it into the server's `plugins` folder and restart the server.

## Commands
- `/caav` - command help
- `/area edit` - get a selector axe, then right-click four corner blocks in order
- `/vnpc create <name>` - spawn a custom villager
- `/vnpc edit <name>` - open the trade editor
- `/vnpc claim <name>` - claim stored payment items

## Trade editor
1. Hold the payment item stack in your main hand; its stack amount becomes the price quantity.
2. Run `/vnpc edit <name>` and click **Set Cost**.
3. Hold the reward item stack in your main hand.
4. Reopen `/vnpc edit <name>` and click **Set Reward**.
5. Reopen the editor and click **Save Trade**.
6. Right-click the custom villager to open its shop. Click a listed reward to trade.

## Notes
- Protected areas apply to the polygon enclosed by the four corners and extend vertically through the world.
- Regular players cannot break/place blocks or interact with blocks in protected areas; entity interaction is allowed.
- Admin permission: `caav.admin` (default OP). General command permission: `caav.use`.
- This is source code; compile with Maven against the Spigot 1.12.2 API. Test on a backup server before production use.
