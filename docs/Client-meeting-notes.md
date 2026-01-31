---
weight: 3
---

# Client meeting notes

## 03-09-2025

### Current Status

- Issues exist with application stability (crashing)
- Nursing department currently has no working version to demonstrate to students

### Project Priorities

- Create a stable, workable application for instructor laptops
- Fix the existing shock simulation (sweating, increased heart rate)
- Develop the digestive system visualization ("from mouth to bottom")

## 23-10-2025

### Progress

- Fixed crashes and created a stable version of the anatomical model application that can be downloaded and used on laptops.
- The application works well with all original features including simulation, layers, and slicer functionality.
- Visual elements remain unchanged; improvements focused on stability.
- Current model includes bones, heart, lungs, and veins.

### Issues and Future Development

- The quiz mode built by previous students doesn't work at all.
- The digestive system currently exists as one whole part, not separated into individual components.
- Client prioritizes developing the digestive system functionality for teaching purposes.

### Digestive System Requirements

- Need animations showing the complete digestive process from mouth to exit.
- Functionality to demonstrate processes like vomiting.
- Separate organs rather than one whole piece, similar to how the heart and lungs work.

## 26-11-2025

### Current Progress

- Stable version of the application delivered to the client
- Model was demonstrated during an open day alongside a VR experience  
- Positive feedback from board members and prospective students  
- Strong interest in continued development and storytelling behind the project

### Issues

- Application startup is unreliable, sometimes failing to launch
(It was because the client didn't know how to unzip the file. Make sure you explain clearly to them how to do it)
- High battery consumption on laptops
- Blood flow visualization issue (oxygen-rich vs oxygen-poor) reported by students, but not yet diagnosed due to startup failures

### Client Priorities

- **Unified interaction layout**
  - All model interactions grouped in one place
  - Consistent interaction model across layers
  - Reduced mouse travel
  - Clear separation between:
    - Interaction controls
    - Informational content

- **Simulation Controls**
  - Future-ready parameter design required:
    - Blood pressure  
    - Temperature  
    - Heart rate  
    - Oxygen  

- **Digestion Simulation**
  - Phase selection is desirable for the future:
    - Mouth → stomach  
    - Stomach → small intestine  
    - Small intestine → large intestine  
  - Not a current priority (focus remains on anatomy phase)

- **User Interaction Improvements**
  - Requested **auto-zoom** functionality:
    - Select anatomical structure → automatic focus
    - Zoom to region or isolate structure
    - Support teaching and in-class explanation

## 08-12-2025

### Design Recommendations

- **Auto-zoom improvement:** When clicking on an anatomical structure, the system should:
    - Automatically zoom in without extra clicks
    - Use the entire screen space (full "real estate")
    - Make other body parts transparent but still visible for context
    - When showing joints, include parts of connected bones to provide context

- **UI/UX principles:**
    - Reduce mouse travel and clicks to improve user experience
    - Eliminate hidden features that users might not discover
    - Distinguish between dashboard (information display) and control panel (interactive elements)
    - Use consistent placement of controls across all models
    - Improve hover indicators for rotation controls

- **Performance considerations:**
    - The hospital environment is visually attractive but reduces performance and adds little value
    - Performance should be prioritized over visual elements that don't add functional value

### Technical Considerations

- **Database structure:**
    - The current database may need restructuring to be future-proof
    - The system should allow adding new elements without requiring database rewrites
    - The database should properly represent anatomical relationships (e.g., differentiating between a body part and an anatomical area)
    - The structure is more critical than the content at this stage

- **Feature recommendations:**
    - Consider removing the slicer feature and replace with isolation of body parts
    - Implement context-aware zooming based on anatomical relationships
    - Design with future requirements in mind (e.g., adding symptoms or pathology)