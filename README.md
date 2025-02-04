# dashboard
python data analysis
import pandas as pd
import plotly.express as px
import dash
from dash import dcc, html
from dash.dependencies import Input, Output
import base64
import io

# Initialize the Dash app
app = dash.Dash(__name__)

# App layout
app.layout = html.Div([
    html.Div([
        html.H1("📊 Data Analysis Dashboard", style={'textAlign': 'center', 'color': '#2c3e50'}),
        html.Hr(),
        dcc.Upload(
            id='upload-data',
            children=html.Div(['Drag and Drop or ', html.A('Select Files')]),
            style={
                'width': '100%',
                'height': '60px',
                'lineHeight': '60px',
                'borderWidth': '2px',
                'borderStyle': 'dashed',
                'borderRadius': '10px',
                'textAlign': 'center',
                'margin': '20px',
                'backgroundColor': '#ecf0f1'
            },
            multiple=False
        ),
        html.Div(id='output-data-upload')
    ], style={'maxWidth': '900px', 'margin': 'auto'})
])

# Function to parse uploaded data
def parse_contents(contents, filename):
    try:
        content_type, content_string = contents.split(',')
        decoded = base64.b64decode(content_string)
        df = pd.read_csv(io.StringIO(decoded.decode('utf-8')))

        numeric_columns = df.select_dtypes(include='number').columns.tolist()
        categorical_columns = df.select_dtypes(include='object').columns.tolist()

        graph_section = []

        if numeric_columns:
            graph_section.append(
                html.Div([
                    html.H4('Select Numeric Column for Histogram:'),
                    dcc.Dropdown(
                        id='histogram-dropdown',
                        options=[{'label': col, 'value': col} for col in numeric_columns],
                        value=numeric_columns[0],
                        style={'marginBottom': '10px'}
                    ),
                    dcc.Graph(id='histogram-graph')
                ], style={'backgroundColor': '#f9f9f9', 'padding': '15px', 'borderRadius': '8px', 'margin': '10px 0'})
            )

        if len(numeric_columns) >= 2:
            graph_section.append(
                html.Div([
                    html.H4('Scatter Plot (X vs Y):'),
                    dcc.Dropdown(
                        id='scatter-x-dropdown',
                        options=[{'label': col, 'value': col} for col in numeric_columns],
                        value=numeric_columns[0],
                        style={'marginBottom': '5px'}
                    ),
                    dcc.Dropdown(
                        id='scatter-y-dropdown',
                        options=[{'label': col, 'value': col} for col in numeric_columns],
                        value=numeric_columns[1],
                        style={'marginBottom': '10px'}
                    ),
                    dcc.Graph(id='scatter-graph')
                ], style={'backgroundColor': '#f0f8ff', 'padding': '15px', 'borderRadius': '8px', 'margin': '10px 0'})
            )

        summary = html.Div([
            html.H3(f'Data from {filename}', style={'color': '#2980b9'}),
            html.H4('📋 Data Preview:'),
            html.Div([
                html.Pre(df.head().to_string(index=False), style={'backgroundColor': '#fafafa', 'padding': '10px'})
            ], style={'overflowX': 'auto'}),

            html.H4('📊 Data Summary:'),
            html.Pre(df.describe().to_string(), style={'backgroundColor': '#f4f6f7', 'padding': '10px'}),
        ] + graph_section)

        return summary, df.to_dict('records')
    except Exception as e:
        return html.Div([
            html.H4("Error processing file:"),
            html.Pre(str(e), style={'color': 'red'})
        ]), []

# Callback to handle file upload
@app.callback(
    [Output('output-data-upload', 'children'),
     Output('output-data-upload', 'data')],
    [Input('upload-data', 'contents'),
     Input('upload-data', 'filename')]
)
def update_output(contents, filename):
    if contents is not None:
        return parse_contents(contents, filename)
    return '', []

# Histogram Callback
@app.callback(
    Output('histogram-graph', 'figure'),
    [Input('histogram-dropdown', 'value'),
     Input('output-data-upload', 'data')]
)
def update_histogram(selected_column, data):
    if data and selected_column:
        df = pd.DataFrame(data)
        fig = px.histogram(df, x=selected_column, title=f'Histogram of {selected_column}',
                           color_discrete_sequence=['#3498db'])
        fig.update_layout(bargap=0.2, template='plotly_white')
        return fig
    return {}

# Scatter Plot Callback
@app.callback(
    Output('scatter-graph', 'figure'),
    [Input('scatter-x-dropdown', 'value'),
     Input('scatter-y-dropdown', 'value'),
     Input('output-data-upload', 'data')]
)
def update_scatter(x_col, y_col, data):
    if data and x_col and y_col:
        df = pd.DataFrame(data)
        fig = px.scatter(df, x=x_col, y=y_col, title=f'Scatter Plot: {x_col} vs {y_col}',
                         color_discrete_sequence=['#e74c3c'])
        fig.update_layout(template='plotly_white')
        return fig
    return {}

# Run the app
if __name__ == '__main__':
    app.run_server(debug=False, port=8051, use_reloader=False, host='0.0.0.0')

