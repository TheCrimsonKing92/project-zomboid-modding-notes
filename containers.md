# Containers

## Get a friendly name

### As used in the loot panel
A user-"unfriendly" name can be obtained with `local containerType = container:getType()`.
The loot panel gets a friendly name by looking up the translated container title: `local containerName = getTextOrNull("IGUI_ContainerTitle_" .. containerType) or "");`