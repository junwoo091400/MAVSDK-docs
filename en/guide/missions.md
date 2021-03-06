# Missions

The Mission API (plugin) allows you to create, upload, download, import from *QGroundControl*, run, pause, restart, jump to item in, and track missions. 
Missions can have multiple "mission items", each which may specify a position, altitude, fly-through behaviour, camera action, gimbal position, and the speed to use when traveling to the next position.

Missions are *managed* though the [Mission](../api_reference/classmavsdk_1_1_mission.md) class, which communicates with the vehicle to upload mission information and run, pause, track the mission progress etc. 
The mission that is uploaded to the vehicle is defined as a vector of [MissionItem](../api_reference/classmavsdk_1_1_mission_item.md) objects.


## Supported Mission Commands {#supported_mission_commands}

The [MissionItem](../api_reference/classmavsdk_1_1_mission_item.md) class abstracts a small but useful subset of the mission commands supported by PX4 (and the MAVLink specification):

The supported set is:
* [MAV_CMD_NAV_WAYPOINT](https://mavlink.io/en/messages/common.html#MAV_CMD_NAV_WAYPOINT)
* [MAV_CMD_DO_CHANGE_SPEED](https://mavlink.io/en/messages/common.html#MAV_CMD_DO_CHANGE_SPEED)
* [MAV_CMD_DO_MOUNT_CONTROL](https://mavlink.io/en/messages/common.html#MAV_CMD_DO_MOUNT_CONTROL)
* [MAV_CMD_IMAGE_START_CAPTURE](https://mavlink.io/en/messages/common.html#MAV_CMD_IMAGE_START_CAPTURE)
* [MAV_CMD_IMAGE_STOP_CAPTURE](https://mavlink.io/en/messages/common.html#MAV_CMD_IMAGE_STOP_CAPTURE)
* [MAV_CMD_VIDEO_START_CAPTURE](https://mavlink.io/en/messages/common.html#MAV_CMD_VIDEO_START_CAPTURE)
* [MAV_CMD_VIDEO_STOP_CAPTURE](https://mavlink.io/en/messages/common.html#MAV_CMD_VIDEO_STOP_CAPTURE)
* [MAV_CMD_NAV_LOITER_TIME](https://mavlink.io/en/messages/common.html#MAV_CMD_NAV_LOITER_TIME)

Additionally, the following commands are supported only for mission import/download (there are no corresponding `MissionItem` methods):
* [MAV_CMD_NAV_LAND](https://mavlink.io/en/messages/common.html#MAV_CMD_NAV_LAND)
* [MAV_CMD_NAV_TAKEOFF](https://mavlink.io/en/messages/common.html#MAV_CMD_NAV_TAKEOFF)

> **Tip** The Mission API does not (at time of writing) provide explicit functionality to "repeat", takeoff, return to land etc. 
> The SDK provides some omitted functionality through the [Action](../guide/taking_off_landing.md) API.


## Create the Plugin

> **Tip** `Mission` objects are created in the same way as other MAVSDK plugins.
  General instructions are provided in the topic: [Using Plugins](../guide/using_plugins.md).

The main steps are:

1. Link the plugin library into your application. 
   Do this by adding `mavsdk_mission` to the `target_link_libraries` section of the app's *cmake* build definition file

   ```cmake
   target_link_libraries(your_application_name
     mavsdk
     ...
     mavsdk_mission
     ...
   )
   ```
1. [Create a connection](../guide/connections.md) to a `system`. 
   For example (basic code without error checking):
   ```
   #include <mavsdk/mavsdk.h>
   Mavsdk dc;
   ConnectionResult conn_result = dc.add_udp_connection();
   // Wait for the system to connect via heartbeat
   while (!dc.is_connected()) {
      sleep_for(seconds(1));
   }
   // System got discovered.
   System &system = dc.system();
   ```
1. Create a shared pointer to an instance of `Mission` instantiated with the `system`: 
   ```
   #include <mavsdk/plugins/mission/mission.h>
   auto mission = std::make_shared<Mission>(system);
   ```

The `mission` pointer can then used to access the plugin API (as shown in the following sections).


## Defining a Mission

A mission must be defined as a vector of [MissionItem](../api_reference/classmavsdk_1_1_mission_item.md) objects as shown below:
```cpp
std::vector<std::shared_ptr<MissionItem>> mission_items;
```

You can create as many `MissionItem` objects as you like and use `std_vector::push_back()` to add them to the back of the mission item vector. 
The example below shows how to create and add a `MissionItem`, that just sets the target position (using [set_position()](../api_reference/classmavsdk_1_1_mission_item.md#classmavsdk_1_1_mission_item_1abe17a24cf27fa0b6d632f39f8d7d15f3)).
```cpp
// Create MissionItem and set its position
std::shared_ptr<MissionItem> new_item(new MissionItem());
new_item->set_position(47.40 /*latitude_deg*/, 8.5455360114574432 /*longitude_deg*/);

// Add new_item to the vector
mission_items.push_back(new_item);
```

The example below shows how you might set the other options on a second `MissionItem` (and add it to the mission).

```cpp
std::shared_ptr<MissionItem> new_item2(new MissionItem());
new_item2->set_position(47.50 /*latitude_deg*/, 8.5455360114574432 /*longitude_deg*/);
new_item2->set_relative_altitude(2.0f);
new_item2->set_speed(5.0f /* m/s */);
new_item2->set_fly_through(true);
new_item2->set_gimbal_pitch_and_yaw(20.0f /*pitch*/, 60.0f /*yaw degrees*/);
new_item2->set_camera_action(MissionItem::CameraAction::TAKE_PHOTO);
    
//Add new_item2 to the vector
mission_items.push_back(new_item2);
```

> **Note** The autopilot has sensible default values for the attributes. 
  If you do set a value (e.g. the desired speed) then it will be the default for the remainder of the mission.

<span></span>
> **Note** There are also getter methods for querying the current value of `MissionItem` attributes. 
  The default values of most fields are `NaN` (which means they are ignored/not sent).

The mission (`mission_items`) can then be uploaded as shown in the section [Uploading a Mission](#uploading_mission) below.

### Convenience Function

The [Fly Mission](../examples/fly_mission.md) uses a convenience function to create `MissionItem` objects. 
Using this approach you have to specify every attribute for every mission item, whether or not the value is actually used.

The definition and use of this function is shown below:

```cpp
std::shared_ptr<MissionItem> make_mission_item(double latitude_deg,
                                               double longitude_deg,
                                               float relative_altitude_m,
                                               float speed_m_s,
                                               bool is_fly_through,
                                               float gimbal_pitch_deg,
                                               float gimbal_yaw_deg,
                                               MissionItem::CameraAction camera_action)
{
    std::shared_ptr<MissionItem> new_item(new MissionItem());
    new_item->set_position(latitude_deg, longitude_deg);
    new_item->set_relative_altitude(relative_altitude_m);
    new_item->set_speed(speed_m_s);
    new_item->set_fly_through(is_fly_through);
    new_item->set_gimbal_pitch_and_yaw(gimbal_pitch_deg, gimbal_yaw_deg);
    new_item->set_camera_action(camera_action);
    return new_item;
}


mission_items.push_back(
    make_mission_item(47.398170327054473,
                      8.5456490218639658,
                      10.0f, 5.0f, false,
                      20.0f, 60.0f,
                      MissionItem::CameraAction::NONE));
```

## Import a Mission from a QGC Plan {#import_qgc_plan}

`Mission` allows you to import a mission from a *QGroundControl* plan (the imported mission can then be uploaded to a vehicle).

> **Note** To export a mission plan from the *QGroundControl* use the [Sync Tool](https://docs.qgroundcontrol.com/en/PlanView/PlanView.html#file) (**Plan View > Sync Tool**, and then select **Save to File**).

The mission is imported using the static [import_qgroundcontrol_mission](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1aa0a937d18e9ce2c4f346aa9ee8a1a6d9) method.
The method will fail with an error if the plan file cannot be found, cannot be parsed, or if it contains mission items that are [not supported](#supported_mission_commands).

The code fragment below shows how to import mission items from a plan:
```cpp
std::string qgc_plan = "file_path_to_some_qgroundcontrol.plan"
Mission::mission_items_t mission_items;
Mission::Result import_res = Mission::import_qgroundcontrol_mission(mission_items, qgc_plan);
```

> **Tip** [Example:Fly QGC Plan Mission](../examples/fly_mission_qgc_plan.md) provides a working example with error checking.

The mission (`mission_items`) can then be uploaded as shown in the section [Uploading a Mission](#uploading_mission) below.


## Uploading a Mission {#uploading_mission}

Use [Mission::upload_mission_async()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1ad6117cf541865249900201fe6305f5a1) to upload the mission defined in the previous section.

The example below shows how this is done, using promises to wait on the result.

```cpp
// ... declare and populate the mission vector: mission_items

{
    auto prom = std::make_shared<std::promise<Mission::Result>>();
    auto future_result = prom->get_future();
    mission->upload_mission_async(
    mission_items, [prom](Mission::Result result) {
        prom->set_value(result);
    });

    const Mission::Result result = future_result.get();
    if (result != Mission::Result::SUCCESS) {
        std::cout << "Mission upload failed (" << Mission::result_str(result) << "), exiting." << std::endl;
        return 1;
    }
    std::cout << "Mission uploaded." << std::endl;
}
```

## Starting/Pausing Missions 

Start or resume a paused mission using [Mission::start_mission_async()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1af0a6404b7af085759bc6dad518eea525).
The vehicle must already have a mission (the mission need not have been uploaded using the SDK).

The code fragment below shows how this is done, using promises to wait on the result.

```cpp
{
    auto prom = std::make_shared<std::promise<Mission::Result>>();
    auto future_result = prom->get_future();
    mission->start_mission_async(
    [prom](Mission::Result result) {
        prom->set_value(result);
    });

    const Mission::Result result = future_result.get(); //Wait on result
    if (result != Mission::Result::SUCCESS) {
        std::cout << "Mission start failed (" << Mission::result_str(result) << "), exiting." << std::endl;
        return 1;
    }
    std::cout << "Started mission." << std::endl;
}
```

To pause a mission use [Mission::pause_mission_async()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1a39a0434bb2e731ccca7c0f2d579b0ee8). 
The code is almost exactly the same as for starting a mission:

```cpp
{
    auto prom = std::make_shared<std::promise<Mission::Result>>();
    auto future_result = prom->get_future();

    std::cout << "Pausing mission..." << std::endl;
    mission->pause_mission_async(
    [prom](Mission::Result result) {
        prom->set_value(result);
    });

    const Mission::Result result = future_result.get();
    if (result != Mission::Result::SUCCESS) {
        std::cout << "Failed to pause mission (" << Mission::result_str(result) << ")" << std::endl;
    } else {
        std::cout << "Mission paused." << std::endl;
    }
}
```

## Monitoring Progress

Asynchronously monitor progress using [Mission::subscribe_progress()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1aa882278b493a2390ca0b3cf7027b3a01), 
which receives a regular callback with the current `MissionItem` number and the total number of items.

The code fragment just takes a lambda function that reports the current status. 

```cpp
mission->subscribe_progress( [](int current, int total) {
       std::cout << "Mission status update: " << current << " / " << total << std::endl;
    });
```

> **Note** The mission is complete when `current == total`.

The following synchronous methods are also available for checking mission progress:
* [mission_finished()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1a5861a4c6325ca1eeb1cc1d37004a7f61) - Checks if mission has been finished.
* [current_mission_item()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1a25afa022a4206bd1055a692937f9b916) - Returns the current mission item index.
* [total_mission_items()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1a97c07cbcf5e9c54d5776cd86e4228311) - Gets the total number of items.

> **Note** The mission is (also) complete when `current_mission_item()` == `total_mission_items()`.


## Taking Off, Landing, Returning

If using a copter or VTOL vehicle then PX4 will automatically takeoff when it is armed and a mission is started (even without a takeoff mission item). 
For Fixed Wing vehicles the vehicle must be launched before starting a mission.

At time of writing the Mission API does not provide takeoff, land or "return to launch" `MissionItems`. 
If required you can instead use the appropriate commands in the [Action](../guide/taking_off_landing.md) class.

<!-- Update if we get new mission items -->

## Downloading Missions

Use [Mission::download_mission_async()](../api_reference/classmavsdk_1_1_mission.md#classmavsdk_1_1_mission_1a23173385eef570b9802402fc047104c1) to download a mission from the vehicle.
The mission is downloaded as a vector of [MissionItem](../api_reference/classmavsdk_1_1_mission_item.md) objects, that you can then view or manipulate as required.

> **Note** Mission download will fail if the mission contains a command that is outside the [supported set](#supported_mission_commands). 
> Missions created using *QGroundControl* are not guaranteed to successfully download! 

The code fragment below shows how to download a mission:

```cpp
{
    std::cout << "Downloading mission." << std::endl;

    // Wrap download_mission_async() function using std::future.
    struct PromiseResult {
        Mission::Result mission_result;
        std::vector<std::shared_ptr<MissionItem>> mission_items;
    };

    auto prom = std::make_shared<std::promise<PromiseResult>>();
    auto future = prom->get_future();

    mission->download_mission_async(
        [prom](Mission::Result result, std::vector<std::shared_ptr<MissionItem>> mission_items_downloaded) {
            PromiseResult promise_result {};
            promise_result.mission_result = result;
            promise_result.mission_items = mission_items_downloaded;
            prom->set_value(promise_result);
    });

    PromiseResult promise_result = future.get();

    if (promise_result.mission_result != Mission::Result::SUCCESS) {
        std::cout << "Mission download failed (" << Mission::result_str(promise_result.mission_result) 
            << "), exiting." << std::endl;
        return 1;
    }

    std::cout << "Mission downloaded (MissionItems: " 
        << promise_result.mission_items.size() 
        << ")" << std::endl;
}
```


## Further Information

* [Mission Flight Mode](https://docs.px4.io/master/en/flight_modes/mission.html) (PX4 User Guide)
* [Example:Fly Mission](../examples/fly_mission.md)
* [Example:Fly QGC Plan Mission](../examples/fly_mission_qgc_plan.md)
* Integration tests:
  * [mission.cpp](https://github.com/mavlink/MAVSDK/blob/{{ book.github_branch }}/src/integration_tests/mission.cpp)
  * [mission_change_speed.cpp](https://github.com/mavlink/MAVSDK/blob/{{ book.github_branch }}/src/integration_tests/mission_change_speed.cpp)
  * [mission_survey.cpp](https://github.com/mavlink/MAVSDK/blob/{{ book.github_branch }}/src/integration_tests/mission_survey.cpp)
* Unit Tests:
  * [mission_import_qgc_test.cpp](https://github.com/mavlink/MAVSDK/blob/{{ book.github_branch }}/src/plugins/mission/mission_import_qgc_test.cpp)