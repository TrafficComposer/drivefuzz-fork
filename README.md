# **Note**: This repository is a fork of [drivefuzz](https://gitlab.com/s3lab-code/public/drivefuzz) and is not the official version.

Some resources that the original DriveFuzz depends on have become outdated.
The authors of [TrafficComposer](https://github.com/TrafficComposer/TrafficComposer) have modified the DriveFuzz repository to address these issues.

Summary of modifications:

- Updated `carla/Util/BuildTools/Setup.sh`.

- Changed the default port in `src/config.py` Line 47: `self.sim_tm_port = 8000` -> `self.sim_tm_port = 5000`.

# DriveFuzz

DriveFuzz is a feedback-driven fuzzing framework for testing autonomous
driving systems end-to-end. DriveFuzz automatically generates safety-critical
driving scenarios in which an autonomous driving system misbehaves.

Our paper, "DriveFuzz: Discovering Autonomous Driving Bugs through Driving
Quality-Guided Fuzzing" will be published at ACM CCS 2022 in November.

Please visit [DriveFuzz website](https://drivefuzz.s3lab.io)
for more information and demonstration.

## Testing environment

The following environment was used to perform the testing for this
project:

* Hardware
  * CPU: Intel Xeon Gold 5218
  * GPU: GeForce RTX 2080 Ti (x8)
  * RAM: 192 GB
* OS & SW
  * Ubuntu 18.04 with Linux 5.4.0-104-generic
    * This is strictly required by Autoware, one of our test targets
  * Python 3.6.9

In addition, CARLA (the simulator we use) requires the following:
(based on `https://carla.readthedocs.io/en/0.9.10/build_linux/`)
  * Ubuntu 18.04
  * 30GB disk space
  * An adequate GPU
  * Two TCP ports and good internet connection


## Installation

Please follow `INSTALL.md` to install all required packages.


## Running DriveFuzz

0. Clone this repository to `~/drivefuzz`

    ```
    $ git clone https://gitlab.com/s3lab-code/public/drivefuzz.git ~/drivefuzz
    ```

1. Run carla simulator

    First check if carla is already running:
    ```
    $ docker ps | grep carla-$USER
    ```

    If not, run Carla container by executing:
    ```
    $ cd ~/drivefuzz
    $ ./run_carla.sh
    ```

2. Prepare environment

    ```
    $ cd ~/drivefuzz
    $ ./init.sh
    $ source /opt/ros/melodic/setup.bash
    ```

3. Run fuzzing

  * Testing Behavior Agent
    ```
    $ cd ~/drivefuzz/src
    $ ./fuzzer.py -o out-artifact -s seed-artifact -c 5 -m 5 -t behavior --timeout 60 --town 1 --strategy all
    ```

    * Outputs (bugs, mutated test inputs, metadata, ...) will be stored in `out-artifact`
    * Sample seeds are provided in `seed-artifact`
    * Adjust the number of cycles (`-c`) and size of mutated population (`-m`)
      (please refer to Algorithm 1 in the paper)
    * You can try other mutation strategies (congestion, entropy, instability,
      trajectory); (please refer to Section 4.2.4 in the paper)
    * Check `./fuzzer.py --help` for further instructions

    Please note that Behavior Agent has a bug in the controller (that we found and
    reported), which leads to frequent lane invasions and finishing before
    reaching the goal. You can prevent these from triggering the misbehavior
    detector by adding `--no-lane-check` and `--no-other-check` flags, e.g.,

    ```
    $ ./fuzzer.py -o out-artifact -s seed-artifact -c 5 -m 5 -t behavior --timeout 60 --town 1 --strategy all --no-lane-check --no-other-check
    ```

* Testing Autoware
    ```
    $ cd ~/drivefuzz/src
    $ rm -rf out-artifact
    $ ./fuzzer.py -o out-artifact -s seed-artifact -c 5 -m 5 -t autoware --timeout 60 --town 1 --strategy all --sim-port 2000
    ```


## Results

By running DriveFuzz on Autoware and Behavior Agent, we detected the following
bugs shown in the tables. We provide the test cases (including the seed,
metadata, vehicle states, and detected bugs). Videos of the bugs can be found
[here](https://www.youtube.com/channel/UCpCrUiGanDKX-qxj8jcUVGQ).

### Autoware bugs
| Bug # | Description                                                                                   | Test case & Log | Video                                         |
|-------|-----------------------------------------------------------------------------------------------|-----------------|-----------------------------------------------|
| 1     | LiDAR and camera fusion misses small objects on road                                          | logs/bug1.json  | <https://www.youtube.com/watch?v=3o1g4NbBzvI> |
| 2     | Perceives the road ahead as an obstacle at a steep downhill                                   | logs/bug2.json  | <https://www.youtube.com/watch?v=6DWNnOyhWVE> |
| 3     | Fails to semantically tag detected traffic lights and cannot take corresponding actions       | logs/bug3.json  | <https://www.youtube.com/watch?v=_vwiINJHebY> |
| 4     | Fails to semantically tag detected stop signs and cannot take corresponding actions           | logs/bug4.json  | <https://www.youtube.com/watch?v=_vwiINJHebY> |
| 5     | Fails to semantically tag detected speed signs and cannot take corresponding actions          | logs/bug5.json  | <https://www.youtube.com/watch?v=_vwiINJHebY> |
| 6     | Faulty localization of the base frame while turning                                           | logs/bug6.json  | <https://www.youtube.com/watch?v=xgwL2AkGtyY> |
| 7     | Localization error when moving underneath bridges and intersections                           | logs/bug7.json  | <https://www.youtube.com/watch?v=xaj77hqKlG4> |
| 8     | Generates infeasible path if the given goal is unreachable                                    | logs/bug8.json  | <https://www.youtube.com/watch?v=iWlzCRar500> |
| 9     | Generates infeasible path if the goal's orientation is not aligned with lane direction        | logs/bug9.json  | <https://www.youtube.com/watch?v=lUKdf7b_7xY> |
| 10    | Global path starts too far from the vehicle's current location                                | logs/bug10.json | <https://www.youtube.com/watch?v=ZI3aS32QDmw> |
| 11    | Target speed keeps increasing at certain roads, overriding the speed configuration            | logs/bug11.json | <https://www.youtube.com/watch?v=Ms5jmjqo-pw> |
| 12    | Fails to avoid forward collision with a moving object                                         | logs/bug12.json | <https://www.youtube.com/watch?v=CA8rBKqwUt0> |
| 13    | Fails to avoid lateral collision (Autoware perceives the approaching actor before collision)  | logs/bug13.json | <https://www.youtube.com/watch?v=ZnrLI2v9OJU> |
| 14    | Fails to avoid rear-end collision (Autoware perceives the approaching actor before collision) | logs/bug14.json | <https://www.youtube.com/watch?v=kh59yH2k53k> |
| 15    | While turning, ego-vehicle hits an immobile actor partially blocking the intersection         | logs/bug15.json | <https://www.youtube.com/watch?v=RyP-_Puf0Sg> |
| 16    | Ego-vehicle keeps moving after reaching the destination                                       | logs/bug16.json | <https://www.youtube.com/watch?v=gWXF0D4KTQ0> |
| 17    | Fails to handle sharp right turns, driving over curbs                                         | logs/bug17.json | <https://www.youtube.com/watch?v=RtFm3fJ1kCA> |

### Behavior Agent bugs

| Bug # | Description                                                                                             | Test case & Log | Video                                         |
|-------|---------------------------------------------------------------------------------------------------------|-----------------|-----------------------------------------------|
| 18    | Indefinitely stops if an actor vehicle is stopped on a sidewalk                                         | logs/bug18.json | <https://www.youtube.com/watch?v=zLv0yyZlW0I> |
| 19    | Flawed obstacle detection logic; lateral movement of an object is ignored                               | logs/bug19.json | <https://www.youtube.com/watch?v=qRz7aA8LM7E> |
| 20    | Generates inappropriate trajectory when initial position is given within an intersection                | logs/bug20.json | <https://www.youtube.com/watch?v=e1Pwmbeim1w> |
| 21    | Improper lane changing, cutting off and hitting an actor vehicle                                        | logs/bug21.json | <https://www.youtube.com/watch?v=KAeGXxF3us8> |
| 22    | Vehicle indefinitely stops at stop signs as planner treats stop signs as red lights and waits for green | logs/bug22.json | <https://www.youtube.com/watch?v=-qoYl6AGNno> |
| 23    | Vehicle does not preemptively slow down when the speed limit is reduced                                 | logs/bug23.json | <https://www.youtube.com/watch?v=P7FvNKbiwa0> |
| 24    | Always stops too far (> 10 m) from the goal due to improper checking of waypoint queue                  | logs/bug24.json | <https://www.youtube.com/watch?v=fNDhTa6C9cg> |
| 25    | Collision prevention does not work at intersections (only checks if actors are on the same lane)        | logs/bug25.json | <https://www.youtube.com/watch?v=Gtp0mtYSlBE> |
| 26    | Fails to avoid lateral collision (Behavior Agent perceives the approaching actor before collision)      | logs/bug26.json | <https://www.youtube.com/watch?v=geC3hBmY11g> |
| 27    | Fails to avoid rear-end collision (Behavior Agent perceives the approaching actor before collision)     | logs/bug27.json | <https://www.youtube.com/watch?v=gW7vl5MaG1s> |
| 28    | No dynamic replanning; the vehicle does infeasible maneuvers to go back to missed waypoints             | logs/bug28.json | <https://www.youtube.com/watch?v=O9sTu8MilI8> |
| 29    | Keeps over-accelerating to achieve the target speed while slipping, creating jolt back on dry surface   | logs/bug29.json | <https://www.youtube.com/watch?v=gV8asQA6okw> |
| 30    | Motion controller parameters (PID) are poorly tuned, making the vehicle overshoot at turns              | logs/bug30.json | <https://www.youtube.com/watch?v=duHTXJIn2EQ> |

### Carla simulator bugs
| Bug # | Description                                                       | Test case & Log | Video                                         |
|-------|-------------------------------------------------------------------|-----------------|-----------------------------------------------|
| 31    | Simulation does not properly apply control commands               | logs/bug31.json | <https://www.youtube.com/watch?v=BW8VNQNP6fM> |
| 32    | Vector map contains a dead end blocked by objects as a valid lane | logs/bug32.json | <https://www.youtube.com/watch?v=1XjsAOI2ILs> |
| 33    | Occasionally inconsistent simulation result                       | logs/bug33.json | <https://www.youtube.com/watch?v=cTDwj5zqjK0> |


## Troubleshooting - DriveFuzz hanging at "Waiting for Autoware nodes .."

There has been a few questions about this issue, and it is most likely an
issue in the Autoware installation, not DriveFuzz itself.

Upon successful initialization on our hardware,
Autoware created >= 195 ROS topics:
`base_topics (195) + num_actor_vehicles_in_scene (varies) + num_walkers_in_scene (varies)`.
DriveFuzz runs `rostopic list` to check the number of active ROS topics
and waits until the number matches the expected number of topics above.

Therefore, if Autoware is not initialized properly,
DriveFuzz will keep waiting for more ROS topics to pop up
and not proceed to a fuzzing phase.

Potential reasons for the initialization issue include, but are not limited to:
- Mismatch of operating system:
  as documented in [above](#testing-environment), Autoware strictly requires Ubuntu 18.04.
  We haven't had success in running Autoware on other versions of Ubuntu.
- Graphics driver issue:
  We discovered that Autoware is **very** dependent on the Nvidia driver and
  associated CUDA version. I hate to say this, but you might need to match our
  [hardware settings](#testing-environment) in order to fuzz Autoware.
- The number of expected topics is different across systems:
  it seems that the number of ROS topics Autoware create varies across
  different systems. If you think this is the case
  (for example, you can run Autoware standalone and check the number of topics
   it creates when it is working properly),
  you can simply modify `WAIT_AUTOWARE_NUM_TOPICS` in `src/constants.py`
  to the number of topics you found.
  Alternatively, you can make DriveFuzz not wait for Autoware initialization
  by setting `WAIT_AUTOWARE_NUM_TOPICS = 0`. Please note that in this case,
  correct operation of Autoware is not guaranteed.

Again, this is a very annoying issue in Autoware that we also experienced
and we would love to find a graceful solution. If anyone has a workaround
or needs additional assistance,
you are more than welcome to drop me (seulbae@gatech.edu) an email.
Thank you!
