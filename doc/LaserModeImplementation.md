# Laser Mode Implementation Strategy

## 1. Overview

This document outlines two alternative strategies for implementing laser mode support in PrusaSlicer:
1. Strategy A: New Parameter Approach - Using dedicated laser parameters
2. Strategy B: Variable Reuse Approach - Maximum reuse of existing FDM parameters

### 1.1 Key Objectives
- Add laser mode with minimal impact on existing code
- Ensure backward compatibility with FDM functionality
- Maintain code clarity and debuggability
- Support both implementation strategies

## 2. Strategy A: New Parameter Approach

### 2.1 Configuration Changes
Location: `src/libslic3r/PrintConfig.hpp`

Add new configuration options:
```cpp
// Laser mode options
ConfigOptionBool          laser_mode;                  // Whether to use laser mode instead of FDM mode
ConfigOptionFloats        laser_spot_width;           // Width (X/Y) of the laser spot in mm
ConfigOptionFloats        laser_spot_height;          // Height (Z) of the laser spot in mm
```

Rationale:
- Clear separation between FDM and laser parameters
- Explicit control over laser spot dimensions
- Multi-laser support through array structure

### 2.2 Layer Height Validation Changes
Location: `src/libslic3r/Slicing.cpp`

```cpp
coordf_t Slicing::max_layer_height_from_nozzle(const DynamicPrintConfig &print_config, int idx_nozzle)
{
    // Check if we're in laser mode
    if (print_config.has("laser_mode") && print_config.opt_bool("laser_mode", false)) {
        // For laser mode, allow much finer layer heights
        coordf_t laser_height = print_config.opt_float("laser_spot_height", idx_nozzle - 1);
        if (laser_height > 0)
            return laser_height;  // Use full spot height
    }
    
    // Default FDM behavior
    coordf_t max_layer_height = print_config.opt_float("max_layer_height", idx_nozzle - 1);
    coordf_t nozzle_dmr = print_config.opt_float("nozzle_diameter", idx_nozzle - 1);
    return (max_layer_height == 0.) ? (0.75 * nozzle_dmr) : max_layer_height;
}
```

### 2.3 Flow Calculations
Location: `src/libslic3r/Flow.cpp`

```cpp
class Flow {
private:
    bool    m_laser_mode;     // Mode indicator
    float   m_spot_width;     // Laser spot width
    float   m_spot_height;    // Laser spot height
    // ... existing members ...

public:
    double mm3_per_mm() const {
        if (m_laser_mode)
            return m_spot_width * m_spot_height;  // Simple rectangular area
        // Original FDM calculation
        return m_bridge ? 
            (M_PI * m_height * m_height) * 0.25 :
            m_width * m_height - (1.0 - 0.25 * M_PI) * m_height * m_height;
    }
};
```

### 2.4 Flow Class Changes
Location: `src/libslic3r/Flow.hpp`

```cpp
class Flow {
    // New constructor for laser mode
    static Flow new_from_laser_config(
        float spot_width,
        float spot_height,
        bool is_bridge = false
    );

    // Modified existing constructor to handle laser mode
    Flow(float width, float height, float spacing, float nozzle_diameter, 
         bool bridge, bool laser_mode = false, 
         float spot_width = 0, float spot_height = 0);
};
```

### 2.5 Configuration Validation
Location: `src/libslic3r/PrintConfig.cpp`

```cpp
void DynamicPrintConfig::validate()
{
    if (this->opt_bool("laser_mode")) {
        // Validate laser spot dimensions
        const std::vector<double> &spot_widths = this->option<ConfigOptionFloats>("laser_spot_width")->values;
        const std::vector<double> &spot_heights = this->option<ConfigOptionFloats>("laser_spot_height")->values;
        
        for (size_t i = 0; i < spot_widths.size(); ++i) {
            if (spot_widths[i] <= 0)
                throw InvalidSpotDimensionError("Laser spot width must be positive");
            if (spot_heights[i] <= 0)
                throw InvalidSpotDimensionError("Laser spot height must be positive");
        }
    }
}
```

### 2.6 G-Code Generation
Location: `src/libslic3r/GCode/GCodeWriter.hpp`

```cpp
class GCodeWriter {
private:
    bool m_laser_mode;

public:
    void set_laser_mode(bool enabled) { m_laser_mode = enabled; }
    bool laser_mode() const { return m_laser_mode; }
    
    // Modified extrusion methods for laser mode
    std::string extrude_to_xy(const Vec2d &point, double dE, const std::string &comment = "") {
        if (m_laser_mode) {
            // Laser-specific G-code generation
            return laser_move_to_xy(point, comment);
        }
        // Existing FDM code...
    }
    
private:
    std::string laser_move_to_xy(const Vec2d &point, const std::string &comment);
};
```

## 3. Strategy B: Variable Reuse Approach

### 3.1 Configuration Changes
Location: `src/libslic3r/PrintConfig.hpp`

Reuse existing configuration structures:
```cpp
// Reuse nozzle_diameter for laser spot width
ConfigOptionFloats        nozzle_diameter;           // Represents laser spot width in laser mode
// Reuse layer_height for laser spot height
ConfigOptionFloat         layer_height;              // Represents laser spot height in laser mode
// Add only one new option
ConfigOptionBool          laser_mode;                // Switch between FDM and laser mode
```

### 3.2 Flow Class Changes

#### 3.2.1 Variable Reuse Strategy
```cpp
class Flow {
private:
    float   m_width;          // Width for both modes (nozzle/spot width)
    float   m_height;         // Height for both modes (layer/spot height)
    float   m_spacing;        // Line spacing (equals width in laser mode)
    float   m_nozzle_diameter;// Nozzle/spot diameter reference
    bool    m_bridge;         // Unused in laser mode
    bool    m_laser_mode;     // Mode flag
};
```

#### 3.2.2 Semantic Mappings
- FDM Mode:
  - m_width = extrusion width
  - m_height = layer height
  - m_spacing = calculated spacing
  - m_nozzle_diameter = physical nozzle size
- Laser Mode:
  - m_width = spot width
  - m_height = spot height
  - m_spacing = spot width
  - m_nozzle_diameter = spot width reference

#### 3.2.3 Flow Calculations
```cpp
double Flow::mm3_per_mm() const {
    if (m_laser_mode) {
        return m_width * m_height;  // Simple rectangular area
    }
    // Keep existing FDM calculations
    return m_bridge ? 
        (M_PI * m_height * m_height) * 0.25 :
        m_width * m_height - (1.0 - 0.25 * M_PI) * m_height * m_height;
}

float Flow::spacing() const {
    return m_laser_mode ? m_width : m_spacing;
}
```

### 3.4 Flow Calculations
Location: `src/libslic3r/Flow.cpp`

```cpp
float Flow::spacing() const {
    if (m_laser_mode) {
        return m_width;  // In laser mode, spacing equals spot width
    }
    // Original FDM spacing calculation
    return m_bridge ? 
        m_width - m_height * (1. - 0.25 * PI) :  // Bridge spacing
        m_width - m_height * (1. - 0.25 * PI);
}

float Flow::width() const {
    return m_laser_mode ? m_width : // Spot width in laser mode
           m_bridge ? m_height * PI/4 : // Bridge width
           m_width;  // Normal width
}
```

### 3.5 Configuration Validation
Location: `src/libslic3r/PrintConfig.cpp`

```cpp
void DynamicPrintConfig::validate()
{
    if (this->opt_bool("laser_mode")) {
        // In laser mode, nozzle_diameter is used as spot width
        const std::vector<double> &nozzle_diameters = this->option<ConfigOptionFloats>("nozzle_diameter")->values;
        
        for (double d : nozzle_diameters) {
            if (d <= 0)
                throw InvalidSpotDimensionError("Laser spot width (nozzle_diameter) must be positive");
        }
        
        // Layer height becomes spot height
        double layer_height = this->option<ConfigOptionFloat>("layer_height")->value;
        if (layer_height <= 0)
            throw InvalidSpotDimensionError("Laser spot height (layer_height) must be positive");
    } else {
        // Original FDM validation...
    }
}
```

### 3.6 Constructor Chain
Location: `src/libslic3r/Flow.hpp`

```cpp
class Flow {
public:
    // Base constructor with mode awareness
    Flow(float width, float height, float spacing, float nozzle_diameter, 
         bool bridge, bool laser_mode = false)
        : m_width(width)
        , m_height(height)
        , m_spacing(laser_mode ? width : spacing)
        , m_nozzle_diameter(nozzle_diameter)
        , m_bridge(laser_mode ? false : bridge)
        , m_laser_mode(laser_mode)
    {}

    // Static constructors
    static Flow new_from_config(
        FlowRole role,
        const ConfigOptionFloatOrPercent &width_param,
        float nozzle_diameter,
        float height,
        bool bridge,
        bool laser_mode = false
    );
};
```

## 4. Strategy Comparison

### 4.1 Strategy A: New Parameter Approach
Advantages:
- Clear separation of concerns
- Explicit laser parameters
- Easier to extend with laser-specific features
- More intuitive for users

Disadvantages:
- More code changes required
- More parameters to maintain
- Potential for parameter duplication

### 4.2 Strategy B: Variable Reuse Approach
Advantages:
- Minimal code changes
- Reuses existing parameter validation
- Simpler maintenance
- Less memory usage

Disadvantages:
- Less intuitive parameter meanings
- May limit future laser-specific features
- Could cause confusion in multi-material setups

## 5. Implementation Recommendations

### 5.1 Short Term
Implement Strategy B (Variable Reuse) because:
- Faster to implement
- Minimal code changes
- Lower risk of bugs
- Maintains backward compatibility

### 5.2 Long Term
Consider migrating to Strategy A (New Parameters) when:
- Adding more laser-specific features
- Implementing multi-laser support
- Developing advanced laser parameters
- User feedback indicates need for clearer parameters

## 6. Testing Strategy

### 6.1 Common Tests for Both Strategies
1. Backward Compatibility:
   - Verify FDM mode works unchanged
   - Check all existing test cases pass

2. Laser Mode Tests:
   - Verify spot dimensions are respected
   - Check flow calculations
   - Validate spacing logic

3. Integration Tests:
   - Test mode switching
   - Verify parameter inheritance
   - Check multi-tool scenarios

### 6.2 Strategy-Specific Tests

#### Strategy A Tests:
- Validate new laser parameters
- Test multi-laser configurations
- Check parameter independence

#### Strategy B Tests:
- Verify parameter reuse logic
- Test semantic overloading
- Validate backward compatibility

## 7. Future Considerations

### 7.1 Common to Both Strategies
1. Optimization Opportunities:
   - Laser-specific spacing algorithms
   - Power/speed parameters
   - Material-specific adjustments

2. Potential Enhancements:
   - Laser focus control
   - Variable power settings
   - Multiple spot profiles

### 7.2 Strategy-Specific Considerations

#### Strategy A:
- Advanced laser parameter sets
- Multi-laser coordination
- Laser-specific material profiles

#### Strategy B:
- Parameter semantic clarity
- Migration path to explicit parameters
- Multi-material compatibility

## 8. Common Components

### 8.1 GUI Updates
Location: `src/slic3r/GUI/ConfigWizard.cpp`

```cpp
void ConfigWizard::priv::add_page(PageType type)
{
    switch (type) {
    case PageType::PRINTER:
        // Add laser mode checkbox
        auto *laser_mode = new wxCheckBox(parent, wxID_ANY, _(L("Enable Laser Mode")));
        laser_mode->SetToolTip(_(L("Switch between FDM and Laser mode")));
        grid_sizer->Add(laser_mode, 0, wxEXPAND | wxALL, 5);
        break;
    }
}
```

### 8.2 Error Handling
Location: `src/libslic3r/Exceptions.hpp`

```cpp
class LaserModeError : public Slic3r::Exception {
public:
    using Exception::Exception;
};

class InvalidSpotDimensionError : public LaserModeError {
public:
    using LaserModeError::LaserModeError;
};
```

### 8.3 G-Code Generation
Location: `src/libslic3r/GCode/GCodeWriter.cpp`

```cpp
std::string GCodeWriter::laser_move_to_xy(const Vec2d &point, const std::string &comment)
{
    std::string gcode;
    
    // Move to position
    gcode += "G1 ";
    if (point.x() != m_pos.x() || point.y() != m_pos.y()) {
        if (point.x() != m_pos.x())
            gcode += "X" + float_to_string(point.x());
        if (point.y() != m_pos.y())
            gcode += "Y" + float_to_string(point.y());
    }
    
    // Add laser power if needed
    if (m_laser_power > 0)
        gcode += " S" + float_to_string(m_laser_power);
        
    if (!comment.empty())
        gcode += " ; " + comment;
    
    gcode += "\n";
    m_pos = point;
    
    return gcode;
}
```

This document provides a comprehensive comparison of both implementation strategies, allowing for informed decision-making based on project requirements and constraints.
