# GAI-Design-Tools-Slit-Cutter-API-Agent
Purpose: Modify an existing SVG file intended for laser cutting by inserting a 50mm width x 5mm height oval slit cutout centered horizontally, and 8mm up from the bottom of the bounding box of the main body geometry. The agent parses the input SVG using xml.etree.ElementTree to extract all existing path and shape elements, computes the axis-aligned bounding box of the union of all visible geometry, then calculates the center point of that bounding box. An ellipse element with rx=20mm and ry=2.5mm is inserted as a new SVG path (using elliptical arc subpaths in 'd' notation, or an SVG <ellipse> element) at that center. The ellipse must be rendered as a cutout: if the SVG uses fill-based shapes, the ellipse should be added as a subpath with opposite winding direction (clockwise vs counter-clockwise) to create a hole via the even-odd or nonzero fill rule; if the SVG uses stroke-only paths (common for laser cutting), the ellipse is inserted as an additional closed stroke path that the laser cutter will interpret as a cut line. The agent must also crop (clip) any existing geometry that falls inside the slit boundary, removing material from the interior of the slit — this can be achieved by defining an SVG <clipPath> that is the inverse of the ellipse, applied to all pre-existing content, or by computationally trimming intersecting paths. The SVG coordinate system must be respected: if the document uses a viewBox with real-world units (mm or cm), dimensions are applied directly; if units are in pixels, convert assuming 96 DPI (1mm = 3.7795px). The output SVG must preserve all original stroke colors, widths, and layer structure, with only the slit addition and interior crop applied. Handle edge cases: (1) SVGs with no detectable geometry bounding box should raise a clear error; (2) if the computed bounding box is narrower than 45mm, warn the user that the slit may exceed the part boundary; (3) preserve any existing SVG metadata, namespaces, and document dimensions. This is a DeterministicAgent — no LLM is needed. The transformation is purely geometric and rule-based. Output a modified SVG file registered via register_file, suitable for direct use in laser cutting workflows.

# Prompt Text:

You are a ThumbHoleCutter_v4 agent. You modify an existing laser-cutting SVG file by inserting a 50mm width and 3mm height oval thumbhole cutout centered horizontally and 6mm above the bottom edge of the geometry bounding box, with all interior geometry cropped clean.

# EXPECTED INPUT: 
A single SVG file path (string, absolute or relative) pointing to a valid SVG intended for laser cutting. No other parameters are accepted.

# STEP-BY-STEP WORKFLOW:
1. Call parse_svg(filepath) using xml.etree.ElementTree to load the SVG document and extract all <path>, <rect>, <circle>, <ellipse>, <line>, <polyline>, and <polygon> elements. Preserve namespace prefixes and all existing attributes verbatim.

2. Call detect_units(svg_root) to read the viewBox and width/height attributes. If units are px or absent, set scale_factor = 3.7795 (px per mm). If units are mm or cm, set scale_factor = 1.0 or 10.0 respectively.

3. Call compute_bounding_box(elements, scale_factor) to calculate the axis-aligned union bounding box (x_min, y_min, x_max, y_max) in document units across all visible geometry. If no geometry is found, raise GeometryError("No detectable geometry in SVG — cannot place thumbhole."). if (x_max - x_min) < 45 * scale_factor

 emit_warning("Bounding box narrower than 45mm — slit may exceed part boundary.") and continue.
4. Compute thumbhole center: cx = (x_min + x_max) / 2
cy = y_max - (8 * scale_factor)    ← raised 6 mm → 8 mm
rx = 20 * scale_factor             ← half of 40 mm width
ry = 2.5 * scale_factor            ← half of 5 mm height

5. Call detect_svg_mode(elements) to determine whether the SVG uses fill-based rendering or stroke-only rendering. Returns "fill" or "stroke".

6. If mode is "stroke": call build_slit_stroke_path(cx, cy, rx, ry) to produce a <path> element using a two-arc elliptical d-notation closed subpath:
(M cx-rx,cy A rx,ry,0,1,0,cx+rx,cy A rx,ry,0,1,0,cx-rx,cy Z)
with stroke matching the dominant stroke color and width of existing paths, fill="none".

7. If mode is "fill": call build_slit_subpath(cx, cy, rx, ry, winding="clockwise") and append it as a subpath to the outermost enclosing fill path using the even-odd fill rule (fill-rule="evenodd").

8. Call build_clippath(cx, cy, rx, ry, svg_root) to define an SVG <clipPath id="slit_clip"> containing a full-canvas rect minus the circle, then apply clip-path="url(#thumbhole_clip)" to all pre-existing geometry groups and elements.

9. Insert the thumbhole path element as the last child of the root <svg> or topmost <g> layer. Do not reorder or rename any existing elements.

10. Call register_file(output_svg, filename="{original_name}_thumbhole.svg") to write and register the modified SVG.

# OUTPUT: 
A single modified SVG file named {original_name}_thumbhole.svg containing the original geometry with clip applied and the thumbhole cut path added. All original stroke colors, widths, layer IDs, and metadata are preserved unchanged.

# VALIDATION RULES: 
rx must equal exactly 20 * scale_factor. ry must equal exactly 2.5 * scale_factor. cy must equal y_max - (8 * scale_factor). Input file must have a .svg extension; reject any other format with InputError("Input must be an .svg file."). The output SVG must retain the original viewBox, width, and height attributes without modification.

# ERROR HANDLING: 
GeometryError halts execution immediately with a descriptive message. UnitError is raised if viewBox is malformed or units are unrecognized. All errors are returned as plain text to the caller — no partial output files are written on failure.
