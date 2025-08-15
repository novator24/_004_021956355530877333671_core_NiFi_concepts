# 🖥️ NiFi User Interface Guide

The NiFi UI consists of several key areas that enable flow design and management:

## Main UI Components

### 1. Canvas
- **Main workspace** where you design data flows
- **Drag and drop** processors, process groups, and other components
- **Visual connections** between components
- **Zoom and pan** capabilities for large flows

### 2. Component Toolbar
Located at the top of the canvas:
- **Processor**: Add new processors
- **Input Port**: Create input ports for process groups
- **Output Port**: Create output ports for process groups
- **Process Group**: Create containers for organizing components
- **Remote Process Group**: Connect to remote NiFi instances
- **Funnel**: Combine multiple connections
- **Label**: Add documentation to your flow
- **Connection**: Create relationships between components

### 3. Global Menu
Top-right corner of the UI:
- **Summary**: Overview of all components and their status
- **Counter**: View custom counters
- **Bulletin Board**: System messages and alerts
- **Data Provenance**: Track data lineage
- **Controller Settings**: Configure controller services
- **Flow Configuration History**: View changes to the flow
- **Users**: Manage user access (if security is enabled)
- **Policies**: Manage access policies
- **Help**: Documentation and keyboard shortcuts

### 4. Status Bar
Bottom of the UI showing:
- **Active Threads**: Current thread count
- **Queued FlowFiles**: Total FlowFiles in queues
- **Transmitting Remote Process Groups**: Remote connections status
- **Not Transmitting Remote Process Groups**: Inactive remote connections
- **Running Components**: Count of active processors
- **Stopped Components**: Count of inactive processors
- **Invalid Components**: Components with configuration issues
- **Disabled Components**: Manually disabled components

## Navigation and Basic Operations

### Creating Your First Flow

1. **Add a Processor**
   - Drag the processor icon from the toolbar to the canvas
   - Search for "GenerateFlowFile" in the processor dialog
   - Click "Add" to place it on the canvas

2. **Configure the Processor**
   - Right-click the processor and select "Configure"
   - Set properties in the "Properties" tab
   - Adjust scheduling in the "Scheduling" tab
   - Apply changes and close the configuration dialog

3. **Add Another Processor**
   - Add a "LogAttribute" processor to the canvas

4. **Create a Connection**
   - Hover over the GenerateFlowFile processor until connection points appear
   - Drag from the arrow to the LogAttribute processor
   - Select the "success" relationship in the connection dialog

5. **Start the Flow**
   - Select both processors (Ctrl+click or drag selection box)
   - Click the "Start" button (play icon) in the Operate panel

### Processor States and Controls

**Processor States:**
- **Stopped**: Processor is not running (red square icon)
- **Running**: Processor is actively processing data (green play icon)
- **Disabled**: Processor is disabled and cannot be started (disabled icon)
- **Invalid**: Processor has configuration issues (warning icon)

**Processor Controls:**
- **Start**: Begin processing data
- **Stop**: Stop processing (completes current tasks)
- **Configure**: Open configuration dialog
- **View Data Provenance**: See data lineage for this processor
- **View Status History**: See historical performance metrics
- **Copy**: Duplicate the processor
- **Delete**: Remove the processor (must be stopped first)

## Advanced UI Features

### Process Group Management

**Creating Process Groups:**
1. Drag Process Group icon to canvas
2. Double-click to enter the group
3. Build your flow inside the group
4. Use Input/Output ports to connect to parent level

**Process Group Navigation:**
- **Breadcrumb trail**: Shows current location in hierarchy
- **Double-click**: Enter a process group
- **Up arrow**: Return to parent level
- **Home icon**: Return to root level

### Templates

**Creating Templates:**
1. Select processors/process groups to include
2. Right-click and select "Create template"
3. Provide name and description
4. Template is saved for reuse

**Using Templates:**
1. Click template icon in toolbar
2. Select template from list
3. Drag to canvas to instantiate

### Data Provenance

**Accessing Provenance:**
- Main menu → Data Provenance
- Right-click processor → View data provenance
- Click on queue → View data provenance

**Provenance Search:**
```bash
Search Options:
- FlowFile UUID
- Component ID/Name
- Event Type (CREATE, RECEIVE, SEND, etc.)
- Attribute values
- Time range
- File size range
```

**Provenance Events:**
- **CREATE**: FlowFile created
- **RECEIVE**: Data received from external source
- **SEND**: Data sent to external destination
- **CLONE**: FlowFile duplicated
- **JOIN**: Multiple FlowFiles combined
- **SPLIT**: FlowFile divided into multiple
- **MODIFY**: FlowFile content or attributes changed
- **DROP**: FlowFile removed from flow

### Connection Configuration

**Queue Listing:**
- Right-click connection → List queue
- View FlowFiles waiting in queue
- Download, view, or delete individual FlowFiles
- Useful for debugging and data inspection

**Connection Properties:**
```bash
FlowFile Expiration: 0 sec (never expire) to 24 hours
Back Pressure Object Threshold: 10,000 objects
Back Pressure Data Size Threshold: 1 GB
Load Balance Strategy: DO_NOT_LOAD_BALANCE, PARTITION_BY_ATTRIBUTE, ROUND_ROBIN, SINGLE_NODE
```

**Prioritizers:**
- **FirstInFirstOutPrioritizer**: Process in order received (default)
- **NewestFlowFileFirstPrioritizer**: Process newest files first
- **OldestFlowFileFirstPrioritizer**: Process oldest files first
- **PriorityAttributePrioritizer**: Use FlowFile attribute for priority

## UI Customization and Productivity

### Keyboard Shortcuts

**Navigation:**
- **Ctrl + A**: Select all components
- **Ctrl + C**: Copy selected components
- **Ctrl + V**: Paste components
- **Delete**: Delete selected components
- **Space**: Start/stop selected processors
- **Ctrl + F**: Find components
- **Ctrl + Z**: Undo last action

**View Controls:**
- **Ctrl + Mouse Wheel**: Zoom in/out
- **Ctrl + 0**: Fit to screen
- **Ctrl + 1**: Actual size
- **Mouse drag**: Pan around canvas

### Color Coding

**Processor Status Colors:**
- **Gray**: Stopped
- **Green**: Running
- **Red**: Invalid configuration
- **Yellow**: Disabled
- **Blue**: Running with validation warnings

**Connection Status:**
- **Green**: Normal operation
- **Yellow**: Back pressure threshold reached
- **Red**: Back pressure object/size limit reached

### Summary Page

**Accessing Summary:**
- Hamburger menu → Summary
- Shows all components across entire flow

**Summary Information:**
```bash
For each component:
- Name and type
- Run status
- Input/Output statistics
- Tasks and threads
- Processing time
```

**Summary Actions:**
- Start/stop multiple components
- Bulk configuration changes
- System-wide monitoring
- Performance analysis
