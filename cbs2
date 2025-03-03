from flask import Flask, render_template, request, jsonify
import pandas as pd
import requests
import matplotlib.pyplot as plt
import io
import base64

app = Flask(__name__)

# CSV-bestand met woningprijsindex per regio en jaar
CSV_FILE = "woningprijsindex.csv"
CBS_API_URL = "https://opendata.cbs.nl/ODataApi/odata/85773NED/TypedDataSet"

def update_csv():
    try:
        response = requests.get(CBS_API_URL)
        if response.status_code == 200:
            data = response.json()["value"]
            df = pd.DataFrame(data)
            
            # Selecteer alleen relevante kolommen en hernoem ze
            df = df[["Perioden", "RegioS", "Woningtypen", "PrijsindexBestaandeKoopwoningen_1"]]
            df.rename(columns={
                "Perioden": "Jaar", 
                "RegioS": "Regio", 
                "Woningtypen": "Woningtype", 
                "PrijsindexBestaandeKoopwoningen_1": "Prijsindex"
            }, inplace=True)
            
            # Converteer jaartal (eerste vier tekens van de Perioden waarde)
            df["Jaar"] = df["Jaar"].str[:4].astype(int)
            
            # Opslaan naar CSV
            df.to_csv(CSV_FILE, index=False)
            return "Update succesvol!"
        else:
            return "Fout bij ophalen van indexgegevens."
    except Exception as e:
        return f"Fout bij updaten: {str(e)}"

@app.route('/')
def index():
    return render_template("index.html")

@app.route('/bereken', methods=['POST'])
def bereken_waarde():
    try:
        data = request.json
        oude_waarde = float(data['prijs'])
        aankoopjaar = int(data['jaar'])
        regio = data['regio']
        woningtype = data['woningtype']
        
        # Lees CSV met prijsindex gegevens
        df = pd.read_csv(CSV_FILE)
        
        # Zoek de index van het aankoopjaar
        oude_index = df[(df['Jaar'] == aankoopjaar) & (df['Regio'] == regio) & (df['Woningtype'] == woningtype)]['Prijsindex'].values
        
        if len(oude_index) == 0:
            return jsonify({"error": "Geen data beschikbaar voor deze combinatie."})
        
        oude_index = oude_index[0]
        
        # Zoek de meest recente index
        huidige_index = df[(df['Jaar'] == df['Jaar'].max()) & (df['Regio'] == regio) & (df['Woningtype'] == woningtype)]['Prijsindex'].values[0]
        
        # Bereken de geschatte waarde
        nieuwe_waarde = oude_waarde * (huidige_index / oude_index)
        procentuele_verandering = ((nieuwe_waarde - oude_waarde) / oude_waarde) * 100
        
        return jsonify({
            "waarde": f"€{nieuwe_waarde:,.2f}",
            "verandering": f"{procentuele_verandering:.2f}%"
        })
    except Exception as e:
        return jsonify({"error": str(e)})

@app.route('/grafiek')
def toon_grafiek():
    try:
        regio = request.args.get('regio')
        woningtype = request.args.get('woningtype')
        
        df = pd.read_csv(CSV_FILE)
        df_filtr = df[(df['Regio'] == regio) & (df['Woningtype'] == woningtype)]
        
        if df_filtr.empty:
            return "Geen data beschikbaar."
        
        plt.figure(figsize=(8, 5))
        plt.plot(df_filtr['Jaar'], df_filtr['Prijsindex'], marker='o', linestyle='-')
        plt.xlabel("Jaar")
        plt.ylabel("Prijsindex")
        plt.title(f"Prijsontwikkeling in {regio} voor {woningtype}")
        plt.grid(True)
        
        # Opslaan als afbeelding
        img = io.BytesIO()
        plt.savefig(img, format='png')
        img.seek(0)
        plot_url = base64.b64encode(img.getvalue()).decode()
        return f'<img src="data:image/png;base64,{plot_url}"/>'
    except Exception as e:
        return f"Fout bij genereren van grafiek: {str(e)}"

if __name__ == '__main__':
    app.run(debug=True)
