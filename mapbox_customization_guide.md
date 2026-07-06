# Mapbox iOS Navigation SDK Customization Guide

This document explains the custom modifications made to the iOS Mapbox Navigation SDK (`mapbox-navigation-ios` fork) for the **Air Run** project. It serves as a reference for future developers to understand what changes were made, why they were made, and how to maintain them.

---

## 📌 Overview of Modifications

By default, the Mapbox Navigation SDK dynamically formats navigation distances based on the user's device locale (e.g., displaying miles/feet in the US/UK, and kilometers/meters elsewhere). 

For the **Air Run** app, the requirement is to **globally enforce metric units (Kilometers and Meters)** with custom thresholds and rounding rules regardless of the device's default locale.

To achieve this permanently without edits being wiped out by `flutter clean`, we:
1. Forked the official `mapbox-navigation-ios` repository (v2.18.6) to [aniketp0503/mapbox-navigation-ios](https://github.com/aniketp0503/mapbox-navigation-ios).
2. Created a custom branch: `custom-2.18.6`.
3. Linked the Flutter project's `ios/Podfile` to pull Mapbox dependencies directly from this fork branch.

---

## 🛠️ Code Changes

The changes are implemented in the following two core files of the `MapboxCoreNavigation` framework:

### 1. `NavigationSettings.swift`
**File Path in Fork:** `Sources/MapboxCoreNavigation/NavigationSettings.swift`

This file controls the preferred unit system settings of the navigation lifecycle.

* **Forced Kilometers Unit:**
  The `distanceUnit` property has been modified to always return `.kilometer` rather than dynamically checking `Locale.current.measuresDistancesInMetricUnits`.
  ```swift
  // Modified in fork:
  public dynamic var distanceUnit : LengthFormatter.Unit = .kilometer {
      didSet {
          notifyChanged(property: .distanceUnit, value: distanceUnit.rawValue)
      }
  }
  ```

* **Forced Initialization in Extension:**
  The initializer for `LengthFormatter.Unit` mapping from `MeasurementSystem` was modified to ignore the input and always return `.kilometer`.
  ```swift
  extension LengthFormatter.Unit {
      public init(_ measurementSystem: MeasurementSystem) {
          self = .kilometer 
      }
  }
  ```

---

### 2. `DistanceFormatter.swift`
**File Path in Fork:** `Sources/MapboxCoreNavigation/DistanceFormatter.swift`

This class handles formatting of distance measurements and text displayed to the user in navigation banners.

* **0.25 Rounding Increment:**
  The `roundingIncrement` of the internal `measurementFormatter.numberFormatter` is configured to `0.25` for distance displays.
  
* **Forced Kilometer/Meter Conversion:**
  Inside the string formatting functions (`string(from distance:)`, `string(from measurement:)`, and their attributed string counterparts), the conversion logic was adapted to force Metric values even on non-metric (imperial) locales:
  * When `shouldUseMetricSystem` is false, it converts the distance to **Kilometers** instead of Miles.
  * If the converted value is **1 km or less** (`<= 1`), it converts the measurement to **Meters** instead of Feet/Yards.
  
  ```swift
  // Example implementation in string(from measurement:)
  open func string(from measurement: Measurement<UnitLength>) -> String {
      measurementFormatter.numberFormatter.roundingIncrement = 0.25
      
      var localizedMeasurement = measurement.converted(to: .meters)
      let shouldUseMetricSystem = locale.usesMetricSystem
      
      if shouldUseMetricSystem {
          measurementFormatter.unitOptions = [.providedUnit, .naturalScale]
      } else {
          // Imperial locale fallback overridden to output Metric units (km and m)
          measurementFormatter.unitOptions = .providedUnit
          localizedMeasurement.convert(to: .kilometers)
          if localizedMeasurement.value <= 1 {
              localizedMeasurement.convert(to: .meters)
          }
      }
      return measurementFormatter.string(from: localizedMeasurement)
  }
  ```

---

## 🔗 Project Integration (Podfile)

In the main Flutter project, `ios/Podfile` was modified under `target 'Runner'` to download dependencies from our custom fork branch:

```ruby
  # Custom Forked Mapbox Navigation SDK
  pod 'MapboxNavigation', :git => 'https://github.com/aniketp0503/mapbox-navigation-ios.git', :branch => 'custom-2.18.6'
  pod 'MapboxCoreNavigation', :git => 'https://github.com/aniketp0503/mapbox-navigation-ios.git', :branch => 'custom-2.18.6'
```

---

## 🔄 How to Modify or Update in the Future

If you need to make additional changes to the native Mapbox iOS SDK:

1. **Locate the cloned fork:** Go to the folder where you cloned `mapbox-navigation-ios`.
2. **Ensure you are on the correct branch:**
   ```bash
   git checkout custom-2.18.6
   ```
3. **Make your changes, then commit and push to GitHub:**
   ```bash
   git add .
   git commit -m "Your description of new changes"
   git push origin custom-2.18.6
   ```
4. **Pull latest changes in the Flutter project:**
   Go to `air-run-flutter/ios` directory in your terminal and clear the cache so CocoaPods downloads your new commits:
   ```bash
   pod cache clean --all
   pod update MapboxNavigation MapboxCoreNavigation
   ```
