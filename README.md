# Meteorological Stations of Germany

Analysis of meteorological measurements in Germany. Google Cloud. Python + SQL = SQLAlchemy.

You can directly watch the animation [here](https://radek-kricek.github.io/pages/hexbin_map.html)

The whole code can be found in the 'meteorological_stations.ipynb' notebook.


## The goal

1. Create a single data frame from a set of meteorological measurements in TXT files.
2. Create two tables in a database using SQLAlchemy.
3. Extract relevant information using SQLAlchemy.
4. Create an interactive animated hexagonal map of temperature in Germany.


## Data

The data (over 7000 files with almost 7.5 GB in total) was downloaded from the website of the [European Climate Assessment & Dataset](https://www.ecad.eu/dailydata/predefinedseries.php), the 'Daily mean temperature TG' data set.

## Main parts of code

One of the coding challenges was to create a single data frame with measurements. I defined a function for this purpose and used it in a loop.

```
def parse_file(filename):
    df = pd.read_csv(f'../data/ECA_blend_tg/{filename}', skiprows=19)   # first rows not part of the data frame
    df.columns = df.columns.str.lower().str.strip()   # cleaning column names
    df['date'] = pd.to_datetime(df['date'], format = '%Y%m%d')   # date in the correct format
    df = df[df['q_tg'] == 0]   # only relevant observations
    df.pop('souid')   # drop column
    df.pop('q_tg')   # drop column
    return df

with open("../data/ECA_blend_tg/mean_temperature.csv", mode="w", newline='') as file:   # open CSV file for writing
    for filename in tqdm(os.listdir('../data/ECA_blend_tg')):   # check progress, go through all files in the folder
        if 'TG_STAID' in filename:   # use only relevant measurement files
            df = parse_file(filename)   # apply pre-defined parse function
            df.to_csv(file, index=False, header=False)   # write into the open CSV file
```

Here follows an example of creating a table in the database and uploading data. Use of SQLAlchemy.

```
with engine.begin() as conn:
    conn.execute(text("DROP TABLE IF EXISTS mean_temperature CASCADE;"))
        # CASCADE to drop even if it is bound to other tables by foreign keys
    conn.execute(text("""
        CREATE TABLE mean_temperature (
            staid INT,
            date date,
            tg INT
        );
    """))

df = pd.read_csv('../data/mean_temperature.csv')
df.to_sql('mean_temperature', engine, if_exists='append', index=False)   # load data into the created table in database
```

Extract relevant information from two tables using SQL JOIN statement. Specify common column by USING() and filter by WHERE.

```
with engine.begin() as conn:
    result = conn.execute(text("""
    SELECT staid, year, yearly_temp, latitude, longitude
    FROM yearly_mean_temperature
    JOIN stations
    USING(staid)
    WHERE cn='DE' AND year BETWEEN 1950 AND 2022 AND (year+3)%5 = 0
    ;
    """))
    rows = result.all()
```

Plot animated hexagonal map and save it as HTML file.

```
# plot and animate
fig_5y = ff.create_hexbin_mapbox(
    data_frame=df_germany_5y,
    lat='latitude',
    lon='longitude',
    opacity=0.9,
    mapbox_style='carto-darkmatter',   # map layout background
    height=650,
    center={'lat': 51.1657, 'lon': 10.4515},   # center of the map view
    color='yearly_temp',
    nx_hexagon=13,   # width of the data in map as number of hexagons
    zoom=4,   # zoom of the map view
    labels={"color": "<i>t</i><sub>avg</sub> (°C)"},   # labels correct for physicists
    animation_frame='year',  # animation by year
    title = 'Average temperature in Germany'
)

# dynamic captions:
for frame in fig_5y.frames:
    year = frame['name']
    caption = f'{year}'
    # add the caption as an annotation for each frame:
    frame['layout'].update(
        annotations=[
            go.layout.Annotation(
                text=caption,
                showarrow=False,
                x=0.2,
                y=0.95,
                xanchor='left',
                yanchor='top',
                font=dict(size=25, color="white"),
                bgcolor='black',
                opacity=0.8
            )
        ]
    )

pio.write_html(fig_5y, 'hexbin_map.html')   # save as HTML file
fig_5y.show()   # show in the notebook
```
