# Gcode Processor Development

The UGS core library has a flexible gcode processor plugin system. It is designed
as a processing pipeline to convert one line of code at a time by passing it
through multiple **Command Processor** plugins. Some advanced features in UGS,
like the Auto Leveler, take advantage of this feature to inject a special
processor module into the gcode processing pipeline. Other processors are simpler,
such as the **M30Processor** which simply removes unwanted **M30** commands, or the
**CommandLengthProcessor** which causes an error if the final processed line has
too much data for your controller.

This process is configured using a JSON file which holds processor configuration,
order in which processors should appear in the pipeline, and whether or not they
are enabled. All of this is configurable in UGS and UGP in a **Gcode Processor
Configuration** menu.

## Anatomy of a CommandProcessor

The processor interface is simple. One command goes in along with the current state
and a list of output commands come out. A CommandProcessor might discover invalid
input, in which case a GcodeParserException can be thrown for the GcodeParser to
handle.

```java
public interface CommandProcessor {
    /**
     * Given a command and the current state of a program returns a replacement
     * list of commands.
     * @param command Input gcode.
     * @param state State of the gcode parser when the command will run.
     * @return One or more gcode commands to replace the original command with.
     */
    public List<String> processCommand(String command, GcodeState state) throws GcodeParserException;

    /**
     * Returns information about the current command and its configuration.
     * @return 
     */
    public String getHelp();
}
```

## Simple Example

The **CommandLengthProcessor** is one of the simplest examples of a
CommandProcessor.  One thing of interest is that it accepts a length parameter
during configuration. By adding a CommandLengthProcessor to the GcodeParser,
you can ensure the maximum length of commands.

```java
public class CommandLengthProcessor implements CommandProcessor {
    final private int length;
    public CommandLengthProcessor(int length) {
        this.length = length;
    }

    @Override
    public String getHelp() {
        // Global localization helpers are used for the help message.
        return Localization.getString("sender.help.command.length") + "\n" +
                Localization.getString("sender.command.length")
                + ": " + length;
    }

    @Override
    public List<String> processCommand(String command, GcodeState state) throws GcodeParserException {
        if (command.length() > length)
            throw new GcodeParserException("Command '" + command + "' is longer than " + length + " characters.");

        return Collections.singletonList(command);
    }
}
```

## More Complex Examples
The following examples can be found in GitHub, they wont have the full code
included.

### DecimalProcessor
This is only slightly more complicated than the CommandLengthProcessor. It adds
in some config validation by throwing a RuntimeException in the constructor,
which is handled by code which configures the GcodeParser. The **processCommand**
method is then able to truncate any decimals using simple string manipulation.

### FeedOverrideProcessor
Like the **DecimalProcessor**, the **FeedOverrideProcessor** is able to modifie any
**F-Commands** with simple string manipulation.

### LineSplitter
The **LineSplitter** CommandProcessor will actually modify commands by parsing
the gcode and rewriting it. There are several utilities used here to help:

* **GcodeParser.processCommand** - Converts the command string into something
easier to work with.
* **GcodePreprocessorUtils.extractMotion** - Helper to extract all words associated
to movement commands, the remainder should still be sent but may not be sent in
the context of the rewritten command.

The remaining logic checks the length of any `G0` or `G1` commands and converts
them, or returns the original command unmodified.

## Tutorial: Creating a New CommandProcessor

Creating a fully integrated and configurable **CommandProcessor** is simple, but
does touch a number of different files. This tutorial will go over those pieces
in the context of creating a processor named **M3Dweller**.

### Goal
Create a processor which inserts a **Dwell** command (short delay) whenever the
spindle is enabled. This allows any potentially slow setups such as VFD's to
come up to speed. It should be configurable via the **Gcode Processor
Configuration** menu.

### Creating the processor
First you'll notice that the constructor initializes the command which should
be added after our spindle start **M3** command. The locale is set to make sure
comma is not used as a decimal separator:
```java
    private final String dwellCommand;

    public M3Dweller(double dwellDuration) {
        this.dwellCommand = String.format(Locale.ROOT, "G4P%.2f", dwellDuration);
    }
```

The help method simply adds some comments for the settings GUI:
```java
    @Override
    public String getHelp() {
        return "Add a delay after enabling the spindle with \"M3\" commands. \"M3\" must be the only command on the line.";
    }
```

Everything is then pulled together in the **processCommand** method to return an
extra dwell command when **M3** is detected:
```java
    // Contains an M3 not followed by another digit (i.e. M30)
    Pattern m3Pattern = Pattern.compile(".*[mM]3(?!\\d)(\\D.*)?");

    @Override
    public List<String> processCommand(String command, GcodeState state) throws GcodeParserException {
        if (m3Pattern.matcher(command).matches()) {
            return Arrays.asList(command, dwellCommand);
        }
        return Collections.singletonList(command);
    }
```

All at once now:
```java
public class M3Dweller implements CommandProcessor {
    private final String dwellCommand;

    // Contains an M3 not followed by another digit (i.e. M30)
    Pattern m3Pattern = Pattern.compile(".*[mM]3(?!\\d)(\\D.*)?");

    public M3Dweller(double dwellDuration) {
        this.dwellCommand = String.format(Locale.ROOT, "G4P%.2f", dwellDuration);
    }

    @Override
    public List<String> processCommand(String command, GcodeState state) throws GcodeParserException {
        String noComments = GcodePreprocessorUtils.removeComment(command);
        if (m3Pattern.matcher(noComments).matches()) {
            return Arrays.asList(command, dwellCommand);
        }
        return Collections.singletonList(command);
    }

    @Override
    public String getHelp() {
        return "Add a delay after enabling the spindle with \"M3\" commands. \"M3\" must be the only command on the line.";
    }
}
```

### Hooking up the JSON

The json files are stored in `ugs-core/src/resources/firmware_config`, each of
these will be updated to include our new processor. Notice that we have an
argument named duration, and that it is disabled by default:
```json
                "name": "M3Dweller",
                "enabled": false,
                "optional": true,
                "args": {
                    "duration": 2.5
                }
            },{
```

We also increment the version string, which will prompt users that there is a
new configuration file available and they may need to revisit their settings:
```diff
-    "Version": 3,
+    "Version": 4,
```

### JSON Loader
The last step is to update the **CommandProcessorLoader** to include logic that
creates the new M3Dweller when needed. I used one of the other processors as an
example and made sure to make a corresponding entry for the **M3Dweller** in each
of the corresponding code and comment locations:
```java
                case "M3Dweller":
                    double duration = pc.args.get("duration").getAsDouble();
                    p = new M3Dweller(duration);
                    break;
```

There is also a test which should be updated to make sure everything is working:
```diff
+        args = new JsonObject();
+        args.addProperty("duration", 2.5);
+        object = new JsonObject();
+        object.addProperty("name", "M3Dweller");
+        object.add("args", args);
+        array.add(object);
+
         String jsonConfig = array.toString();
         List<CommandProcessor> processors = CommandProcessorLoader.initializeWithProcessors(jsonConfig);
 
-        assertEquals(8, processors.size());
+        assertEquals(9, processors.size());
         assertEquals(ArcExpander.class, processors.get(0).getClass());
         assertEquals(CommentProcessor.class, processors.get(1).getClass());
         assertEquals(DecimalProcessor.class, processors.get(2).getClass());
         assertEquals(FeedOverrideProcessor.class, processors.get(3).getClass());
         assertEquals(M30Processor.class, processors.get(4).getClass());
         assertEquals(PatternRemover.class, processors.get(5).getClass());
         assertEquals(CommandLengthProcessor.class, processors.get(6).getClass());
         assertEquals(WhitespaceProcessor.class, processors.get(7).getClass());
+        assertEquals(M3Dweller.class, processors.get(8).getClass());
```

### Testing
The command processors lend themselves to thorough testing, so we create a new
`M3DwellerTest.java` file in the associated test package and write some tests:
```java
    @Test
    public void testReplaces() throws Exception {
        M3Dweller dweller = new M3Dweller(2.5);
        String command;

        command = "M3";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command,"G4P2.50");

        command = "m3";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command,"G4P2.50");

        command = "M3 S1000";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command,"G4P2.50");

        command = "m3 S1000";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command,"G4P2.50");

        command = "(this is ignored) M3 S1000";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command,"G4P2.50");
    }
    
    @Test
    public void testNoOp() throws Exception {
        M3Dweller dweller = new M3Dweller(2.5);
        String command;
        command = "anything else";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command);

        command = "M30";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command);

        command = "G0 X0 Y0 (definitely not ready to start the spindle with an M3 yet)";
        Assertions.assertThat(dweller.processCommand(command, null)).containsExactly(command);
    }
```

### Localizing

In addition to the help message which may use the global **Localization** helper,
the settings menu will attempt to find something with the same name as in the
JSON file. So we add an entry to 
`./ugs-core/src/resources/MessagesBundle_en_US.properties`, which will later be
pulled into a separate localization service to be localized in other languages.
```diff
 WhitespaceProcessor = Whitespace Remover
+M3Dweller = Spindle start delay
 controller.exception.smoothie.booting = Smoothie has not finished booting.
```

### Conclusion

We have now fully integrated our **M3Dweller** processor into the UGS framework
and users of UGS and UGP can both make use of it.

The raw commit for this feature is found on [github here](https://github.com/winder/Universal-G-Code-Sender/commit/edc9f08e6eb908706f4985bdbcbbb6ffa831b72a).
<center><img src = "../../img/tutorials/gcode_processor_plugin/1.update.png" width="90%" /></center>
<center><img src = "../../img/tutorials/gcode_processor_plugin/2.settings.png" width="90%" /></center>
<center><img src = "../../img/tutorials/gcode_processor_plugin/3.edit.png" width="90%" /></center>
