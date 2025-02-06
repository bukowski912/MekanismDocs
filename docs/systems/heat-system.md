# The Heat System

## Introduction

This document chronicles the design and development of a (relatively) principled heat system. The main reason for it is as a potential resort to the numerous difficulties and shortcomings of the current offering.

The document is split into two main sections: [core concepts](#core-concepts) and [technical details](#technical-details). The former provides some background and an overview, whilst the latter dives deeper into how such a system might be implemented, along with any other miscellaneous concerns.

---

## Preliminaries

*(Feel free to skip to the [next section](#core-concepts) if you either don't care or already know the stuff.)*

### A little bit of thermodynamics

Thermodynamic systems can be measured and categorised in many ways. For now, we are most interested in the basic physical quantities, such as *temperature* (K, &deg;C, &deg;F), *heat energy* (J, kJ), *specific heat capacity* (J/g-K), *heat capacity* (J/K), and *mass* (g, kg). Here's a couple ways they're related:

1. *heat capacity* = *specific heat* &times; *mass*
2. *temperature* = *heat energy* &divide; *heat capacity*
3. *heat energy* = *temperature* &times; *heat capacity*

There are a few other quantities like [*thermal conductivity* and *resistivity*](https://en.wikipedia.org/wiki/Thermal_conductivity_and_resistivity) which are important when modelling heat flow as a function of time, and are used in e.g. [Fourier's law](https://en.wikipedia.org/wiki/Thermal_conduction#Fourier's_law), but are beyond the scope of this document.

### A little bit of simulation

Heating simulations in many ways resemble physics-sims you'll find in most large game engines. Each *thing* (or body) in the system tracks two important variables: *position* and *velocity*. During every *epoch* (or tick) the velocity is cumulatively modified by (perhaps several) applications of impulse force. At the end of each tick, when all is said and done, the velocity is added to the position and cleared, and thus a single iteration of [*chosen simulation method*](https://en.wikipedia.org/wiki/Explicit_and_implicit_methods) is concluded. It is much the same way with a heat-sim, except that *position* and *velocity* are replaced by *stored heat* and *heat flux*, respectively.

---

## Core Concepts

The proposed system works with several fundamental concepts and objects. Here they are briefly defined, with more details given later.

A handy shopping list of components:

- [Heat capacitors](#heat-capacitor)
- [Heat contacts](#heat-contact)
- [Heat manifolds](#heat-manifold)
- [Heat islands](#heat-island)
- Heat handlers (TODO)

### Heat Capacitor

The most basic unit of heat storage. Capacitors store heat in the form of energy (J); they are also ascribed a heat capacity and various thermal properties (*thermals*) such as conductivity and insulation, which govern their behaviour through [heat contacts](#heat-contact). Thermals may be provided for the entire capacitor (directionless) or for any direction representing a side of that capacitor.

### Heat Contact

A building block of inter-[capacitor](#heat-capacitor) thermodynamics. In a nutshell: contacts facilitate heat *traffic* between one or more capacitors. In practice, they serve two overlapping purposes:

- to introduce heat energy into the system;
- to thermally bond a group of capacitors.

### Heat Manifold

The interconnection of a single [heat capacitor](#heat-capacitor) to any other capacitors, via [heat contacts](#heat-contact). Manifolds are convenient for modelling heat at the scale of a multi-faceted device, such as a heater, reactor, boiler, etc. While a device itself may only manage a single capacitor, a manifold for it will manage the connections that capacitor may have to external elements, such as adjacent tiles, the environment, and so on.

### Heat Island

An aggregation of the above components into a single structure, constrained to a single [*network*](https://en.wikipedia.org/wiki/Component_(graph_theory)) of heat-supporting devices. It exists for the most part to optimize the entire simulation (especially at a large scale) and is otherwise not essential to the system.

---

## Technical Details

### Heat Capacitor

For the sake of numerical simulation, each heat capacitor stores a delta `heatToHandle` which accumulates individual transfers of heat energy that take place within a tick. This task is assigned to the `handleHeat()` instance method. By the end of the tick, this quantity is dumped into the total `storedHeat` of the capacitor. This task is assigned to the `updateHeat()` instance method.

Heat capacitors are outlined by the `IHeatCapacitor` interface, and basically implemented by the `BasicHeatCapacitor` class.

### Heat Contact

Although the heat-contact interface is general enough to support any number of degrees of contact, there are currently only two types of contacts defined, which should serve most if not all use cases:

- **monadic** (single-capacitor)&mdash;outlined by the `IMonadicHeatContact` interface;
- **dyadic** (double-capacitor)&mdash;outlined by the `IDyadicHeatContact` interface.

Contacts may be directional, in that each member capacitor may have an associated direction. This is useful in representing the facets of a capacitor, such as the sides of a block/tile containing one.

An example of a **monadic** contact is an *environmental* connection. Here is a sample of the `simulate()` instance method for such a contact:

```java
@Override
public double simulate(long currentTime) {
    double invConduction = Thermals.environmentConduction(source, side);
    double tempToTransfer = (source.getTemperature() - HeatAPI.AMBIENT_TEMP) / invConduction;
    source.handleHeat(-tempToTransfer * source.getHeatCapacity());
    return Math.max(0, tempToTransfer);
}
```

(Notice that only a single capacitor `source` and direction `side` is needed for this type of contact. Also note that the method takes a `currentTime` parameter. This will be explained later.)

An example of a **dyadic** contact is an *adjacent* connection. Here is a sample of the `simulate()` instance method for such a contact:

```java
@Override
public double simulate(long currentTime) {
    double invConduction = Thermals.adjacentConduction(this);
    double tempToTransfer = (second.getTemperature() - first.getTemperature()) / invConduction;
    double heatToTransfer = tempToTransfer * first.getHeatCapacity();
    first.handleHeat(-heatToTransfer);
    second.handleHeat(heatToTransfer);
    return tempToTransfer;
}
```

(Notice that two capacitors \[and implicitly two directions\]&mdash;one which is opposite the other&mdash;are employed for this type of contact.)

An example for the Thermal Evaporation Plant is provided [at the end](#thermal-evaporation-plant).

### Heat Manifold

The design of the heat manifold promotes contact sharing through its methods, which cuts down on memory usage and simplifies some of the simulation concerns. However, as a contact instance may be tracked by multiple manifolds (doing *double-duty*), further concessions may need to be made.

Since it is assumed that no contact be simulated twice in the same way within a tick, manifolds should reasonably have [*idempotent*](https://en.wikipedia.org/wiki/Idempotence) countermeasures. This can be achieved by the addition of a `lastSimulationTime` variable, which (unsurprisingly) keeps track of the game time of the last simulation tick; and upon being requested to simulate again during the same tick, simply does nothing. To aid in this, the `simulate()` method takes a `currentTime` argument, which is obtained by the caller (most likely from `Level#getGameTime()`) and passed through the call chain.

### Heat Island

In the way of optimization, an island can maintain an internal collection of capacitors and interfaces, which it efficiently iterates through every tick, rather than, say, having to peruse every manifold which would in most cases repeat-visit contacts.

---

## Examples

### Thermal Evaporation Plant

Following is a possible implementation of `IMonadicHeatContact` for use by the Thermal Evaporation Plant.

**Disclaimer:** This has in large part been lifted from the existing implementation; there is potential for improvement.

```java
public class EvaporationHeatContact implements IMonadicHeatContact {

    private long lastSimulationTime = -1;
    private final IntSupplier activeSolarsSupplier;
    private final DoubleSupplier ambientTempSupplier;

    public EvaporationHeatContactBasic(IHeatCapacitor heatCapacitor, IntSupplier activeSolarsSupplier, DoubleSupplier ambientTempSupplier) {
        super(heatCapacitor);
        this.activeSolarsSupplier = activeSolarsSupplier;
        this.ambientTempSupplier = ambientTempSupplier;
    }

    protected double getAmbientTemperature() {
        return ambientTempSupplier == null ? HeatAPI.AMBIENT_TEMP : ambientTempSupplier.getAsDouble();
    }

    @Override
    public double simulate(long currentTime) {
        if (currentTime <= lastSimulationTime) {
            return;
        }
        lastSimulationTime = currentTime;
        
        double currentTemp = getCapacitor().getTemperature();
        int activeSolars = activeSolarsSupplier.getAsInt();
        double ambientTemp = getAmbientTemperature();
        double heatCapacity = getCapacitor().getHeatCapacity();
        capacitor.handleHeat(activeSolars * MekanismConfig.general.evaporationSolarMultiplier.get() * heatCapacity);
        if (Math.abs(currentTemp - ambientTemp) < 0.001) {
            capacitor.handleHeat(ambientTemp * heatCapacity - capacitor.getHeat());
        } else {
            double incr = MekanismConfig.general.evaporationHeatDissipation.get() * Math.sqrt(Math.abs(currentTemp - ambientTemp));
            if (currentTemp > ambientTemp) {
                incr = -incr;
            }
            capacitor.handleHeat(heatCapacity * incr);
            if (incr < 0) {
                return -incr;
            }
        }
        return 0;
    }
}
```
