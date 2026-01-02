# RakshaTerrorismPortal
Visualization of terrorism data and extaction
// ============================================================================
// DASHBOARD: Rashtriya Raksha Terrorism Data Portal
// Author: Sourav Dey
// ============================================================================

// ================= TITLE PANEL =================
var titlePart1 = ui.Label('Rashtriya Raksha Terrorism Data Portal', {
  fontWeight: 'bold', 
  fontFamily: 'Calibri',
  fontSize: '19px',
  textAlign: 'center'
});

var tagline = ui.Label('Empowering Security Through Data Visualization', {
  fontStyle: 'italic',
  fontFamily: 'Times New Roman',
  fontSize: '14px',
  textAlign: 'center'
});

var titlePanel = ui.Panel({
  layout: ui.Panel.Layout.flow('vertical'),
  style: {
    width: 'auto',
    textAlign: 'center',
    backgroundColor: 'white',
    padding: '6px 12px',
    border: '1px solid #ccc',
    borderRadius: '6px',
    position: 'top-center',
    margin: '10px'
  }
});
titlePanel.add(titlePart1);
titlePanel.add(tagline);
Map.add(titlePanel);

// ============================================================================
// LOAD SAMPLE + MAP PROPERTIES
// ============================================================================
var sample = Terrorism.first();
sample.propertyNames().evaluate(function(names) {
  print('Table property names:', names);

  function pickKey(keyword) {
    for (var i = 0; i < names.length; i++) {
      var n = names[i];
      if (!n) continue;
      if (String(n).toLowerCase().indexOf(keyword) !== -1) return n;
    }
    return null;
  }

  var dateKey            = pickKey('date')             || 'Date';
  var incidentKey        = pickKey('incident')         || 'Incident Name';
  var descriptionKey     = pickKey('description')      || 'Description';
  var injuredKey         = pickKey('injured')          || 'Injured';
  var killedKey          = pickKey('killed')           || 'Killed';
  var specificLocationKey= pickKey('specific') || pickKey('location') || 'Specific Location';
  var stateKey           = pickKey('state') || pickKey('region') || 'State/Region';
  var accusedKey         = pickKey('accused') || pickKey('accused count') || 'Accused Count';
  var perpetratorKey     = pickKey('perpetrator') || pickKey('group') || 'Perpetrator Group';
  var caseStatusKey      = pickKey('case') || 'Case Status';
  var yearKey            = pickKey('year')             || 'Year';
  var latKey             = pickKey('latitude') || pickKey('lat') || 'Latitude';
  var lonKey             = pickKey('longitude') || pickKey('lon') || 'Longitude';

  print('Using mapped keys:', {
    dateKey: dateKey, incidentKey: incidentKey, descriptionKey: descriptionKey,
    injuredKey: injuredKey, killedKey: killedKey, specificLocationKey: specificLocationKey,
    stateKey: stateKey, accusedKey: accusedKey, perpetratorKey: perpetratorKey,
    caseStatusKey: caseStatusKey, yearKey: yearKey, latKey: latKey, lonKey: lonKey
  });

  // ================= Convert table to points =================
  var points = Terrorism.map(function(feature) {
    var latS = ee.String(feature.get(latKey));
    var lonS = ee.String(feature.get(lonKey));
    var valid = latS.match('^-?[0-9.]+$').length().gt(0)
             .and(lonS.match('^-?[0-9.]+$').length().gt(0));
    return ee.Algorithms.If(valid,
      ee.Feature(ee.Geometry.Point([ee.Number.parse(lonS), ee.Number.parse(latS)]),
                 feature.toDictionary()),
      ee.Feature(null));
  }, true).filter(ee.Filter.notNull([latKey, lonKey]));

  points = points.map(function(f) {
    return f.set({
      'year_str': ee.String(f.get(yearKey)),
      'incident_str': ee.String(f.get(incidentKey))
    });
  });

  Map.addLayer(points.style({color: 'red', pointSize: 4}), {}, 'Terrorism Incidents');
  Map.centerObject(points, 5);

  // ========================================================================
  // SIDEBAR
  // ========================================================================
  var side = ui.Panel({style: {width: '380px', padding: '8px'}});
  side.add(ui.Label('Terrorism Explorer', {fontWeight: 'bold', fontSize: '16px'}));

  var yearSelect = ui.Select({items: [], placeholder: 'Select Year', style: {stretch: 'horizontal'}});
  side.add(ui.Label('Select Year:'));
  side.add(yearSelect);

  var incidentSelect = ui.Select({items: [], placeholder: 'Select Incident', style: {stretch: 'horizontal'}});
  side.add(ui.Label('Select Incident:'));
  side.add(incidentSelect);

  var infoPanel = ui.Panel();
  side.add(ui.Label('Incident Details:', {fontWeight:'bold', margin: '6px 0 0 0'}));
  side.add(infoPanel);

  ui.root.insert(0, side);

  // ========================================================================
  // FUNCTIONS
  // ========================================================================
  function showIncidentDetails(featureObject) {
    infoPanel.clear();
    if (!featureObject) {
      infoPanel.add(ui.Label('No incident info available.'));
      return;
    }
    var props = featureObject.properties || {};
    var dateVal = props[dateKey] || '';
    infoPanel.add(ui.Label(String(dateVal), {fontWeight: 'bold', fontSize: '14px'}));

    var ordered = [
      {k: incidentKey, label: 'Incident Name'},
      {k: descriptionKey, label: 'Description'},
      {k: injuredKey, label: 'Injured'},
      {k: killedKey, label: 'Killed'},
      {k: specificLocationKey, label: 'Specific Location'},
      {k: stateKey, label: 'State/Region'},
      {k: accusedKey, label: 'Accused Count'},
      {k: perpetratorKey, label: 'Perpetrator Group'},
      {k: caseStatusKey, label: 'Case Status'}
    ];
    ordered.forEach(function(item) {
      var v = props[item.k];
      if (v && String(v).trim() !== '' && String(v).toLowerCase() !== 'null') {
        infoPanel.add(ui.Label(item.label + ': ' + v));
      }
    });
  }

  // Populate Year dropdown
  points.aggregate_array('year_str').distinct().sort().evaluate(function(years) {
    years = (years || []).filter(function(y){return y && String(y).trim() !== '';});
    yearSelect.items().reset(years);
  });

  // Year change → populate incidents
  yearSelect.onChange(function(selectedYear) {
    infoPanel.clear();
    incidentSelect.items().reset([]);
    if (!selectedYear) return;
    var filtered = points.filter(ee.Filter.eq('year_str', selectedYear));
    filtered.aggregate_array('incident_str').distinct().sort().evaluate(function(list) {
      list = (list || []).filter(function(x){return x && String(x).trim() !== '';});
      incidentSelect.items().reset(list);
    });
  });

  // Incident change → zoom + show details
  incidentSelect.onChange(function(selectedIncident) {
    infoPanel.clear();
    var selectedYear = yearSelect.getValue();
    if (!selectedYear || !selectedIncident) return;
    var sel = points.filter(ee.Filter.eq('year_str', selectedYear))
                    .filter(ee.Filter.eq('incident_str', selectedIncident));
    sel.first().evaluate(function(f) {
      if (!f) return;
      try {
        var coords = f.geometry.coordinates;
        Map.setCenter(coords[0], coords[1], 10);
      } catch(e){}
      showIncidentDetails(f);
    });
  });

  // Map click → nearest incident
  Map.onClick(function(coords) {
    infoPanel.clear();
    infoPanel.add(ui.Label('Searching nearest incident...'));
    var clicked = ee.Geometry.Point([coords.lon, coords.lat]);
    points.filterBounds(clicked.buffer(1000)).first().evaluate(function(f) {
      infoPanel.clear();
      if (!f) {
        infoPanel.add(ui.Label('No incident found near this location.'));
        return;
      }
      try {
        var coords = f.geometry.coordinates;
        Map.setCenter(coords[0], coords[1], 10);
      } catch(e){}
      showIncidentDetails(f);
    });
  });

}); // end propertyNames evaluate

// ============================================================================
// FOOTER
// ============================================================================
var footer = ui.Label({
  value: 'Author: Sourav Dey | Contact us | Feedback | Suggestion',
  style: {
    position: 'bottom-center',
    padding: '5px',
    backgroundColor: 'rgba(255, 255, 255, 0)',
    fontSize: '14px',
    color: 'blue',
    textDecoration: 'underline',
    margin: '10px'
  },
  targetUrl: 'https://sites.google.com/view/souravdey/'
});
Map.add(footer);
