# Evacuation System Technical Documentation

## System Overview

The Evacuation System is a real-time application that uses WiFi signal triangulation to locate people in a building during an emergency. It determines the optimal evacuation paths based on congestion levels and fire locations, then guides users to the nearest safe exit.

## Architecture

The system follows a modular architecture with clear separation of concerns:

1. **Models**: Data structures and state management
2. **Services**: Core business logic and algorithms
3. **Routes**: API endpoints and WebSocket handlers
4. **Config**: Centralized configuration
5. **Application Factory**: System initialization and setup

### Directory Structure

```
project/
├── app.py                  # Main application entry point
├── config.py               # Configuration settings
├── models/
│   ├── __init__.py
│   ├── building.py         # Building graph and layout
│   └── device.py           # Device tracking
├── services/
│   ├── __init__.py
│   ├── localization.py     # Location services
│   ├── navigation.py       # Pathfinding services
│   └── emergency.py        # Fire and emergency handling
└── routes/
    ├── __init__.py
    ├── api.py              # API endpoints
    └── websocket.py        # Websocket handlers
```

## Execution Flow

### System Initialization

1. `app.py` creates the Flask application
2. Configuration is loaded from `config.py`
3. Models (Building, DeviceManager) are initialized
4. Services (Emergency, etc.) are created with references to models
5. API and WebSocket routes are registered
6. The application starts listening for connections

### Main Workflows

#### 1. Device Registration and Path Calculation

```
Client -> API (/) -> LocalizationService -> Building -> NavigationService -> DeviceManager -> SocketIO -> Client
```

1. Client sends WiFi signal data to the server
2. `LocalizationService` estimates position using triangulation
3. `Building` determines the nearest node to the position
4. `NavigationService` calculates the optimal path to an exit
5. `DeviceManager` tracks the device and its assigned exit
6. All clients are notified of the update via WebSocket

#### 2. Fire Update

```
Client -> API (/fire) -> EmergencyService -> Building -> SocketIO -> Client
```

1. Client sends fire location data
2. `EmergencyService` updates fire nodes in the building
3. Path recalculation is triggered for all devices
4. All clients are notified of the update via WebSocket

#### 3. Device Exit Completion

```
Client -> API (/exit/<device_tag>) -> DeviceManager -> Building -> SocketIO -> Client
```

1. Client reports successful evacuation
2. `DeviceManager` removes the device from tracking
3. `Building` updates congestion levels
4. All clients are notified of the update via WebSocket

## Component Details

### Models

#### Building (`models/building.py`)

Represents the physical structure with rooms, pathways, and exits.

**Key Responsibilities:**
- Store the building graph with nodes and connections
- Track congestion levels for each node
- Monitor fire-affected areas
- Provide nearest node calculations

**State Management:**
- `graph`: Dictionary of nodes and connections
- `node_congestion`: Congestion counters for each node
- `fire_nodes`: Set of nodes affected by fire

**Key Methods:**
- `get_nearest_node(coordinate)`: Find closest node to a position
- `get_blocked_nodes()`: Get all inaccessible nodes
- `update_fire_nodes(nodes)`: Update fire locations

#### DeviceManager (`models/device.py`)

Tracks all user devices in the system and their evacuation paths.

**Key Responsibilities:**
- Store device locations and assigned paths
- Track devices at each exit point
- Manage exit assignments and capacity

**State Management:**
- `devices`: Dictionary of device data
- `active_exits`: Lists of devices assigned to each exit

**Key Methods:**
- `add_device(device_tag, location_data)`: Register or update a device
- `remove_device(device_tag)`: Remove a device after evacuation
- `get_nearest_available_exit(...)`: Find optimal exit for a user

### Services

#### LocalizationService (`services/localization.py`)

Handles position estimation based on WiFi signal data.

**Key Responsibilities:**
- Process WiFi signal data from devices
- Convert signal strength to distance estimates
- Triangulate device positions

**Key Methods:**
- `process_devices(wifi_devices)`: Extract valid router information
- `triangulate_position(devices, distances, weights)`: Calculate position
- `estimate_position(wifi_devices)`: Main positioning method

#### NavigationService (`services/navigation.py`)

Provides pathfinding and routing functionality.

**Key Responsibilities:**
- Find optimal paths between nodes
- Avoid congested areas and fire zones
- Calculate path distances

**Key Methods:**
- `find_path(graph, start, goal, blocked_nodes)`: A* pathfinding algorithm
- `get_path_for_device(...)`: Calculate optimal evacuation path

#### EmergencyService (`services/emergency.py`)

Manages emergency situations and response.

**Key Responsibilities:**
- Update fire locations
- Trigger path recalculations during changing conditions
- Manage emergency response protocols

**Key Methods:**
- `update_fire_nodes(fire_node_list)`: Update fire locations
- `_recalculate_all_paths()`: Recalculate paths for all devices

### Routes

#### API Routes (`routes/api.py`)

HTTP endpoints for the application.

**Key Endpoints:**
- `POST /`: Process device WiFi data and calculate paths
- `POST /fire`: Update fire locations
- `POST /exit/<device_tag>`: Mark device as successfully evacuated
- `GET /update` and `GET /get_updates`: Provide current system state

#### WebSocket Handlers (`routes/websocket.py`)

Real-time communication with clients.

**Key Events:**
- `connect`: Handle new client connections
- `update`: Broadcast system state changes to all clients

### Configuration (`config.py`)

Centralized settings for the application.

**Key Settings:**
- `FIXED_ROUTERS`: Known WiFi router positions for triangulation
- `EXIT_CAPACITY`: Maximum capacity for each exit
- `CONGESTION_THRESHOLD`: Maximum users per node before congestion
